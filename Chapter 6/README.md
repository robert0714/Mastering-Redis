# Reference

https://hub.packtpub.com/scaling-redis-cluster-and-sentinel/

# Approaches to partitioning data
Partitioning data, where keys are divided and assigned to specific instances, is an important strategy for breaking up large databases or datasets that cannot be loaded into any single machine’s available memory. With partitioning, computation and resources are no longer limitations of a single Redis instance but expands and scales to include multiple processor cores and machines running and connecting to other instances through a network interfaces, routers, and adapters to other machines. There are usually three different avenues for partitioning data with Redis: client-side partitioning, proxy assisted partitioning, and query routing. In client-side partitioning, the partitioning logic is contained in the client code that selects the correct partition or Redis node based on either an algorithm or storing extra information, or some combination of the two. With proxy assisted partitioning, Redis clients connect to a proxy middleware that then routes the client’s requests to the correct Redis node. We will be exploring one of the most popular projects that supports this partitioning approach in a later section in this article on Twemproxy. The final implemented avenue for Redis partitioning is query routing, where any client querying a random node in the cluster will be routed to the correct node containing the key; an approach taken in the current implementation of Redis cluster.

## Range partitioning
Often the simplest method to implement a partitioning strategy on either the server or client-side, Range partitioning does require management code and data structures to keep track of what key is assigned to a particular instance. At its core, Range partitioning assigns an incoming key to an instance based on if the key is inside a particular range of values that have been assigned to an instance.

[![](https://www.packtpub.com/sites/default/files/Article-Images/8181_06_01.png)](https://hub.packtpub.com/scaling-redis-cluster-and-sentinel/)

A simplified version of a range partitioning in Redis is to start with a defined number of Redis instances. In this following example, we will arbitrarily pick five running Redis instances and assign a range of ID to each instance. We will store a bitstring for each instance with a bit flipped to 1 for all the IDs that are in the instance’s assigned range. When a new key is created using a global increment, we will check each partition with a *GETBIT* call to see if the new id is in the assigned range for that partition and if it is, the key will be stored in the Redis instance for that partition.

Here is a short Python function, *set_range_partitions*, that sets the bits for our partitions:
```
def set_range_partitions(datastore, partitions=5, size=20000):
   for i in range(0, partitions):
       key = "partition:{}".format(i+1)
       start = i*size + 1
       end = start + size
       for offset in range(start, end):
           datastore.setbit(key, offset, 1)
```

Now ,we'll import the Redis module for Python , instantiate a StrictRedis class :

```
>>> import redis
>>> tea_datastore = redis.StrictRedis()
>>> set_range_partitions(tea_datastore)
>>> 

```

We now have five partition keys that store the bitstrings for their range of keys. Because these keys do not take up much memory, we can store a copy on each of our five Redis instances. We’ll first examine *partition:1*:

```
127.0.0.1:6379> BITCOUNT partition:1
(integer) 20000
127.0.0.1:6379> GETBIT partition:1 1
(integer) 1
127.0.0.1:6379> GETBIT partition:1 20001
(integer) 0
```
As we would expect, the size of *partition:1* is 20,000. Its first flipped bit starts at offset 1 and the command is as follows:

```
127.0.0.1:6379> BITCOUNT partition:5
(integer) 20000
127.0.0.1:6379> GETBIT partition:5 80000
(integer) 1
127.0.0.1:6379> GETBIT partition:5 99999
(integer) 1
```
To calculate which partition a new key will be stored at is a matter of doing a bitstring operation on the stored partition keys. The easiest method will be to do *GETBIT* with the new key’s numeric id and see if it is set to 1 for each of the partition:

```
127.0.0.1:6379> GETBIT partition:1 568
(integer) 1
127.0.0.1:6379> GETBIT partition:2 568
(integer) 0
127.0.0.1:6379> GETBIT partition:3 568
(integer) 0
127.0.0.1:6379> GETBIT partition:4 568
(integer) 0
127.0.0.1:6379> GETBIT partition:5 568
(integer) 0
```
We will then use *partition:1* for our key *interesting-key:568*:
```
127.0.0.1:6379> SET interesting-key:568 "Some data"
OK
```
For the second key with an id of 83697, we will repeat the process of checking each partition bitstring (the first three GETBIT checks are omitted).

```
127.0.0.1:6379> GETBIT partition:4 83687
(integer) 0
127.0.0.1:6379> GETBIT partition:5 83687
(integer) 1
```

The second key is stored in the fifth running Redis instance of our ad-hoc cluster and is running on port 6382 which we connect to with a running a new Redis-cli session and connecting to that instance to set our second key:

```
127.0.0.1:6382> SET interesting-key:83697 "Another key with info"
OK
```
There are both positive and negative aspects when using the range partition approach to sharding a dataset. Conceptually, range partitioning is the easiest to comprehend and implement. However, as you can see, we do incur overhead costs, both in tracking the partitions with Redis data structures and custom client code to manage both the key assignment to the partition, as well as in retrieval and updating of the keys from the collection of running Redis instances.

## List partitioning
Similar to range partitioning, list partitioning is where a partition is assigned a list of values and if the Redis key has one of the values in a list, it is assigned to that partition. To illustrate list partitioning, we’ll start with a simple telephone application that stores the phone numbers from across the United States into one of the three running Redis instances. What key is assigned to which of the three running Redis instances will be based on a list of area codes assigned to each instance.
[![](https://www.packtpub.com/sites/default/files/Article-Images/8181_06_02.png)](https://hub.packtpub.com/scaling-redis-cluster-and-sentinel/)

Like Range partitioning, using List partitioning requires intermediary data structures to support the assignment and tracking of the keys in the datastore. In this case, we will populate three hashes, with each hash having the area code as the field and the Geographic Area (country or the US state) as the value store at the area code. Because area codes are not necessarily consecutive numbers, we cannot use range partitioning.

Back to our example, we will first open a tab-delimited text file containing the area codes, extract the area code and geographic name from each row, and then assign the first 106 area codes to partition one, the next 106 area codes to partition two, and finally the last 107 area codes to partition three. We will save the three area_code:partition:{id} Redis keys to our first Redis instance that also functions as the first partition.


```
def assign_codes_to_partitions(filename, datastore):
   with open(filename) as area_codes_file:
      area_codes = area_codes_file.readlines()
      area_code_shard1 = "area_code:partition:1"
      area_code_shard2 = "area_code:partition:2"
      area_code_shard3 = "area_code:partition:3"
      for i, row in enumerate(area_codes):
         fields = row.split("\t")
         code = fields[0]
         geo_name = fields[1]
         if i < 106:
            slot= area_code_shard1
         elif i >= 106 and i < 212:
            slot = area_code_shard2
         else:
            slot = area_code_shard3
         datastore.hset(slot, code, geo_name.strip())
```
Now ,we'll import the Redis module for Python , instantiate a StrictRedis class :

```
>>> import redis
>>> tea_datastore = redis.StrictRedis()
>>> assign_codes_to_partitions("/vagrant/conf/hadoop-1/area-codes.txt",tea_datastore)
>>> 
```

To confirm that the first Redis node has three keys and the size of each hash is what we expect, use a Redis-cli session:

```
127.0.0.1:6379> DBSIZE
(integer) 3
127.0.0.1:6379> HLEN area_code:partition:1
(integer) 106
127.0.0.1:6379> HLEN area_code:partition:2
(integer) 106
127.0.0.1:6379> HLEN area_code:partition:3
(integer) 107
```
We will need a second function that takes a phone number, a list of values (name, address, and mobile or land line), and a list of nodes in our cluster; looks up the area codes to get which node to save the phone number hash to; and then saves the values to the sharded Redis instance node:

```
def save_phone_number(phone, values, cluster):
   area_code = phone[0:3]
   if cluster[0].hexists("area_code:partition:1", area_code):
      shard = cluster[0]
   elif cluster[0].hexists("area_code:partition:2", area_code):
      shard = cluster[1]
   else:
      shard = cluster[2]
   shard.hmset(phone, values)
```
Testing our first phone number from our interactive Python shell, we will execute the save_phone_number Python function as follows:

```
>>> save_phone_number(
       "719 555 1212",
       {"name": "Jeremy Nelson",
         "type": "Mobile"},
       cluster)
```
Based on the list partition, the phone number “719 555 1212” is saved as a hash in node 3 of our ad-hoc cluster.

```
127.0.0.1:6379> HEXISTS area_code:partition:2 719
(integer) 1
```
We can confirm by opening a second terminal window to a Redis-cli session with our third node and retrieving all the fields and values from our “719 555 1212” key:

```
$ ~/redis/redis-cli -p 6381
127.0.0.1:6381> hgetall "719 555 1212"
1) "type"
2) "Mobile"
3) "name"
4) "Jeremy Nelson"
```
With this set-up, we should be able to distribute a large number of phone numbers across our three shards. What is missing from our sharding strategy is the means to easily add additional shards as our dataset grows. Because our algorithm is based on the distribution of area codes, we cannot just add more nodes as needed without restructuring our lists. Our strategy also assumes that North American phone numbers are equally distributed among the area codes, with new area codes being added when the existing number of area codes in a region by the North American Numbering Plan Administration (NANPA).

## Hash partitioning
In hash partitioning, a hash algorithm calculates what shard a key is assigned to the datastore. A typical hash function calculates a value from a key and performs a modulo operation on the value based on the number of shards or instances available in the datastore. In a 2011 blog post, titled Redis Presharding5, Salvatore Sanfilippo outlines a basic and simple hashing algorithm that takes a Redis key, hashes it with something such as the SHA1 or CRC, and then does a modulo operation to calculate a location or node to store the key in. Sanfilippo encourages an approach of running many different instances of Redis when creating a cluster. He used 128 Redis instances in his example for hashing the Redis keys on the client-side.

[![](https://www.packtpub.com/sites/default/files/Article-Images/8181_06_03.png)](https://hub.packtpub.com/scaling-redis-cluster-and-sentinel/)

In the Java language, hashing is widely used with a required hashCode() method for classes that create a single 32-bit signed hash value that digests the data stored in a class instance. In one example of using a Java client and Redis for hash sharding, a key made up of an e-mail address is routed to a Redis instance in a cluster through by calling the Java hashCode() method of the email string and storing the key in an email bucket8.

## Composite partitioning

In a composite partitioning strategy, keys are partitioned by different combinations of range, list, and hash partitioning. Redis cluster uses a form of composite partitioning called consistent hashing that combines the features of hash and list partitioning to calculate what instance any particular key is assigned based to, called a hash slot, is on the CRC16 hash algorithm and the computation of a modulo using 16384. The specific algorithm used by Redis cluster to calculate the hash slots for a key is simply the cyclic redundancy check (CRC) using a polynomial length of 17-bits or CRC16.

In the Redis cluster implementation, a minimal three node Redis cluster assigns these hash slots with the first master having hash slots 0 to 5500, the second master node being assigned hash slots 5501 to 11,000, and the third master having the remainder of 11,001 to 16,384 hash slots. The actual hash slot for the key is calculated as the modulo of CRC16 of the key divided by 16384:

```
HASH_SLOT = CRC16(key) modulo 16384
```
The official Redis Cluster specification provides a reference implementation of CRC16 XModem4 that is also available in the Redis source code directory in the crc15.c code file. We can see the hash slot allocation in action if we connect to a running Redis cluster using our Redis-cli with the -c parameter:
```
127.0.0.1:9001> SET book:1 "Mason and Dixon"
OK
127.0.0.1:9001> SET book:2 "Centennial"
-> Redirected to slot [12948] located at 127.0.0.1:9003
OK
```

Our first key, book:1, the hash slot is calculated as crc16(“book:1”) modulo 16384 is 759. In this particular cluster, the master node residing at port 9001 is allocated hash slots 0 to 5500 so with client issuing the SET command stays on the same node. In our second key, book:2, the hash slot is calculated to be crc16(“book:2”) modulo 16384 to be hash slot 12938 which is allocated to the master node at 9003. Instead of doing this hash slot calculation manually, Redis provides a convenient command CLUSTER KEYSLOT that will do this calculation for you:

```
127.0.0.1:9001> CLUSTER KEYSLOT book:1
(integer) 759
127.0.0.1:9001> CLUSTER KEYSLOT book:2
(integer) 12948
```

## Key hash tags

An important exception to the Redis cluster’s standard hash slot allocation discussed previously is the use of hash tags in the Redis key string that restricts the calculation of the
hash slot to just the hash tag. In a Redis key, the hash tags are the characters contained between the first occurrence of the opening brace “{” and the closing brace “}”. This forces the keys to reside in the same hash slot and Redis node in the cluster. This is important because Redis cluster only offers limited support for the multi-key commands while still completely supporting the core Redis commands for the entire cluster. If multi-key commands are needed, all the keys must reside on the same node so that the hash slot calculation can be restricted to just the hash tag. We can test this by returning to our Redis-cli session by first trying to issue a MSET command with keys that do not use hash tags:

```
127.0.0.1:9001> MSET book:3 "Shogun" book:4 "Gone Fishin"

(error) CROSSSLOT Keys in request don't hash to the same slot
```

This is because the hash slot for “book:3” is calculated as crc16(“book:3”) modulo 16384 is 8885 and the hash slot for “book:4” is crc16(“book:3”) modulo 16384 is 4690. Now we’ll try the same command but use hash tag “{book}”:

```
127.0.0.1:9001> MSET {book}:3 "Shogun" {book}:4 "Gone Fishin"OK
```
In this example, we want both the book keys to reside on the same node, so we created the {book} hash tag and issued two SET commands; the first redirected us to the first node because “book” hash slot, crc16(“book”) modulo 16384, is 1337 and the second key resides on the same node that we can confirm by:

```
127.0.0.1:9001> keys *

1) "{book}:3"

2) "book:1"

3) "{book}:4"
```

## Twemproxy

Twemproxy is an open-source project released by Twitter for creating a caching proxy between a client and backend, made up of either Memecache or Redis instances. Twemproxy separates the client calls, in our case any suitable Redis client, from the datastore backend through the use of an intermediary middleware that then implements a sharding strategy based on your preferences that are set in a configuration YAML file. Twemproxy supports twelve different hash functions, including md5, crc16, two versions of crc32, and four variants of the Fowler-Noll-Vo (fnv) among others with the default being fnv1a_64 hash functions.

With Twemproxy being a C program like Redis, the steps to get Twemproxy running first requires either downloading a source tarball or cloning the repository with Git:

```
$ tar xvf nutcracker-0.4.1.tar.gz
$ cd nutcracker-0.4.1/
$ ./configure
$ make
$ sudo make install
```
Before running Twemproxy, we will need to update and configure the proxy to use Redis, and map our running Redis instances as Twemproxy’s backend cache servers.


[![](https://www.packtpub.com/sites/default/files/Article-Images/8181_06_04.png)](https://hub.packtpub.com/scaling-redis-cluster-and-sentinel/)

## Testing Twemproxy with Linked Data Fragments Server

To simplify our testing of Twemproxy with the Linked Data Fragments Server, we will run four Redis instances, using two Redis instances as the master nodes for our Linked Data cache that replicate to the remaining two Redis instances running as the slave nodes. In the new api.py Python module, a new class for a Triple REST endpoint is implemented with two methods: an on_get method for HTTP GET call that returns a serialized JSON of a simple RDF graph of the triple stored at the key in this syntax of {subject sha1}:{predicate sha1}:{object sha1}, and an on_post method for creating a new triple based on the sha1 digests of the subject, predicate, and object and then storing an 1 integer as a value. In the client code, if the triple key is found to exist, then a JSON Linked Data representation of the triple is generated by first splicing the key into its three digest values for the subject, predicate, and object, then retrieving those values held at the sha1 digest keys, and finally, constructing the return JSON string on the fly.

Comparing the performance of the Linked Data Fragments Server involved the creation of test data sets made up of BIBFRAME-based RDF graphs from two sources:

*  Library of Congress MARC records for all the records matching the terms “Mark Twain” and “Bible”
*  All MARC21 records of the most popular material at Colorado College’s library based on the number of checkouts

Altogether, the two datasets represented a total of over 50,000 distinct graphs made up of over 5,000,000 individual triples.

We will extend and continue to isolate the Redis-based code in our project by first creating a cache directory and moving our cache.py to the new directory, renaming it as aioredis.py. We will then create a new Python module in this same directory and call it twemproxy.py.

To begin our testing, we’ll first need to modify Twemproxy’s yaml configuration file located at nutcracker-0.4.1/conf/nutcracker.yml. We’ll be creating a simplified configuration with a single server pool, alpha, for our Redis nodes running on ports 6379 through 6383. Here is the yaml configuration for alpha:

```
alpha:

listen: 127.0.0.1:22121

hash: fnv1a_64

distribution: ketama

auto_eject_hosts: true

redis: true

server_retry_timeout: 2000

server_failure_limit: 1

servers:

     - 127.0.0.1:6379:1

     - 127.0.0.1:6380:1

     - 127.0.0.1:6381:1

     - 127.0.0.1:6382:1
```
To connect to alpha, we will use the Twemproxy port 22121. Under the servers setting, the Redis instances are listed and mapped to the remaining ports. After launching four Redis instances, having been sure to specify separate ports and rdb filenames for each Redis instance, we will open another command line window and launch Twemproxy:

```
$ ./src/nutcracker

[2015-08-17 06:30:52.957] nc.c:187 nutcracker-0.4.1 built for Darwin 14.0.0 x86_64 started on pid 626

[2015-08-17 06:30:52.958] nc.c:192 run, rabbit run / dig that hole, forget the sun / and when at last the work is done / don't sit down / it's time to dig another one

With Twemproxy running, we can connect to port 22121 with our Redis-cli and issue commands:

$ redis/src/redis-cli -p 22121
```
To use our Lua scripts in our Redis instances, we’ll open a Python command-line and loop through all four of the running Redis instances and load add_get_triple.lua into each instance:

```
>>> import redis

>>> with open("/linked-data-fragments/redis/add_get_triple.lua") as fo:add_get_triple = fo.read()

>>> cluster = []

>>> for port in range(6379, 6383):

      instance = redis.StrictRedis(port=port)

      instance.script_load(add_get_triple)

      cluster.append(instance)
```
Switching back to the running Redis-cli session that is connected to Twemproxy and calling EVALSHA with the sha1 hash of add_get_hash results in:

```
127.0.0.1:22121> EVALSHA a5bb6a5952e578bdd2ddd9ede268ab28c6b90eb4 3 http://example.com/book/1 http://schema.org/name "Origins Reconsidered"

"2c866521408acafb64b0e67d17822260d68aadde:30cd0bd17373373839fb3a0ffaa6bba51a17ba6c:574dbf58ad0e51382993cadec21742ae4de5aef8"
```
While Twemproxy evaluated the Lua script and returned the triple composite key made up of the three keys we sent as KEYS when issuing EVALSHA command. However, if we try to retrieve the sha1 key of 2c866521408acafb64b0e67d17822260d68aadde, we get back a nil value in our Redis-cli session:

```
127.0.0.1:22121> GET 2c866521408acafb64b0e67d17822260d68aadde
(nil)
```
So what happened? Because our Lua script populates the sha1 keys for each subject, predicate, and object in our RDF triple, it bypasses Twemproxy even though if we connect directly to our first Redis instance, we can confirm the three keys have been set in only one instance and have not been proxied:

```
127.0.0.1:6379> KEYS *
1) "30cd0bd17373373839fb3a0ffaa6bba51a17ba6c"
2) "2c866521408acafb64b0e67d17822260d68aadde"
3) "574dbf58ad0e51382993cadec21742ae4de5aef8"
127.0.0.1:6379> MGET 2c866521408acafb64b0e67d17822260d68aadde 30cd0bd17373373839fb3a0ffaa6bba51a17ba6c 574dbf58ad0e51382993cadec21742ae4de5aef8
1) "http://example.com/book/1"
2) "http://schema.org/name"
3) "Origins Reconsidered"
```
Using Twemproxy in the Linked Data Fragments Server means that the current Lua scripts for creating and populating the RDF triples is not possible; therefore the logic that exists in the Lua scripts would need to be added to the twemproxy.py module. Since this logic was added in the original implementation of the Redis cache but removed when Lua scripting was implemented in the project, we’ll add the sha1 hashing logic back to the twemproxy.py module. This illustrates an important point about using Twemproxy and Redis in your project – all interactions between your client code and your cache MUST be run through the proxy and not through direct writes to the Redis instances themselves.

[![](https://www.packtpub.com/sites/default/files/Article-Images/8181_06_05.png)](https://hub.packtpub.com/scaling-redis-cluster-and-sentinel/)

After updating twemproxy.py with the additional code to create our sha1 hashes for each triple, we can then retest the performance of Twemproxy. First, we will create a new Lua script, get_triple, that takes a triple and returns a string JSON-LD representation of the triple:

```
local subject_sha1, predicate_sha1, object_sha1 = split(KEYS[1], ":")

local output = '[{"@id": "'

output = output..redis.pcall('get', subject_sha1_)..'",'

output = output..redis.pcall('get', predicate_sha1)..'":[{'

local object = redis.pcall('get', object_sha1)

if string.sub(object,1,string.len("http")) == 'http' then

   output = output..'"@id": "'

else

   output = output..'"@value": "'

end

output = output..'"'..object..'"}]}]'

return output
```
Next, we will load our Colorado College MARC21 test record test into our Twemproxy that is running with four Redis instances. After the test records are loaded, we will connect to each of the Redis instances with Redis-cli to determine the size and amount of memory being used in each of the instances:

```
127.0.0.1:6379> DBSIZE
(integer) 903287
127.0.0.1:6379> INFO memory
# Memory
used_memory:176939920
used_memory_human:168.74M
127.0.0.1:6380> DBSIZE
(integer) 836812
127.0.0.1:6380> INFO memory
# Memory
used_memory:164487104
used_memory_human:156.87M
127.0.0.1:6381> DBSIZE
(integer) 942231
127.0.0.1:6381> INFO memory
# Memory
used_memory:184067520
used_memory_human:175.54M
127.0.0.1:6382> DBSIZE
(integer) 879448
127.0.0.1:6382> INFO memory
# Memory
used_memory:172414320
used_memory_human:164.43M
```
Our Twemproxy Linked Data Fragments has a total of 3,561,778 separate keys with a total memory used between the four instances being 665.58 MB.

## Summary
The biggest change to Redis with the release of the 3.x series is the inclusion of a working, stable, and production-ready Redis Cluster. Redis Cluster is the preferred method of scaling and splitting your data among the different Redis instances running on separate machines. While Redis Cluster implements one method of hashing incoming keys through the use of a composite partitioning method and that combines features from hash and range partitioning, there are other options to scale your data through the use of client-side partitioning methods a few which were illustrated in this article. We also examined a popular open-source alternative for sharding and partitioning data from Twitter called Twemproxy, that provides an intermediary proxy that handles the hash and assignment logic between the application and the Redis instance backends. We then turned back and examined in detail some of the features and functionality of Redis Cluster including its resharding, failover, replacing, and upgrading options that allow for long-running Redis Clusters handling large volumes of data. Finally, this article introduced some advanced usage of Redis Sentinel to monitor a range of different Redis application setups by using examples from earlier in the article.
