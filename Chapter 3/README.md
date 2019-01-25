# 32-bit Redis

page.49

python2 
```
apt install python-pip
pip install pymarc
pip install redis
apt-get install python-matplotlib


>>> import random
>>> import redis
>>> import uuid
>>> INSTANCE64 = redis.StrictRedis()
>>> INSTANCE32 = redis.StrictRedis(host="localhost",port=6380)

>>>def test_redis_32k_64k():
    for i in range(100000):
      key = "uuid:{}".format(i)
      value = str(uuid.uuid4())
      INSTANCE32.set(key, value)
      INSTANCE64.set(key, value)

>>> test_redis_32k_64k()
```

python3
```
apt install python3-pip
pip3 install pymarc
pip3 install redis
apt-get install python-matplotlib


>>> import random
>>> import redis
>>> import uuid
>>> INSTANCE64 = redis.StrictRedis()
>>> INSTANCE32 = redis.StrictRedis(host="localhost",port=6380)

>>>def test_redis_32k_64k():
    for i in range(100000):
      key = "uuid:{}".format(i)
      value = str(uuid.uuid4())
      INSTANCE32.set(key, value)
      INSTANCE64.set(key, value)

>>> test_redis_32k_64k()
```
Redis
```
127.0.0.1:6379> INFO memory
# Memory
used_memory:13108680
used_memory_human:12.50M
used_memory_rss:16564224
used_memory_rss_human:15.80M
used_memory_peak:13108680
used_memory_peak_human:12.50M
used_memory_peak_perc:100.00%
used_memory_overhead:5897568
used_memory_startup:782504
used_memory_dataset:7211112
used_memory_dataset_perc:58.50%
total_system_memory:2090106880
total_system_memory_human:1.95G
used_memory_lua:37888
used_memory_lua_human:37.00K
maxmemory:0
maxmemory_human:0B
maxmemory_policy:noeviction
mem_fragmentation_ratio:1.26
mem_allocator:jemalloc-3.6.0
active_defrag_running:0
lazyfree_pending_objects:0
127.0.0.1:6379> HGETALL user
(empty list or set)
127.0.0.1:6379>  hgetall user
(empty list or set)
127.0.0.1:6379> SCAN 0
1) "106496"
2)  1) "uuid:28914"
    2) "uuid:9424"
    3) "uuid:34926"
    4) "uuid:52885"
    5) "uuid:80817"
    6) "uuid:39154"
    7) "uuid:60166"
    8) "uuid:2278"
    9) "uuid:43839"
   10) "uuid:63823"
127.0.0.1:6379> SCAN 1
1) "69633"
2)  1) "uuid:89597"
    2) "uuid:28607"
    3) "uuid:99550"
    4) "uuid:736"
    5) "uuid:5419"
    6) "uuid:30853"
    7) "uuid:72674"
    8) "uuid:14124"
    9) "uuid:45707"
   10) "uuid:41475"
   11) "uuid:45141"
   12) "uuid:57061"
127.0.0.1:6379> SCAN 1000
1) "41960"
2)  1) "uuid:50315"
    2) "uuid:91243"
    3) "uuid:47639"
    4) "uuid:60877"
    5) "uuid:32024"
    6) "uuid:36812"
    7) "uuid:8710"
    8) "uuid:81210"
    9) "uuid:39951"
   10) "uuid:18614"
127.0.0.1:6379> scan 7
1) "36871"
2)  1) "uuid:69172"
    2) "uuid:95486"
    3) "uuid:69394"
    4) "uuid:91452"
    5) "uuid:78821"
    6) "uuid:27434"
    7) "uuid:7197"
    8) "uuid:97108"
    9) "uuid:16032"
   10) "uuid:72628"

```
# Key expiration
page 53
```
def create_tea(datastore, name, time, size):
  # Increment and save global counter
  tea_counter = datastore.incr("global/teas")
  tea_key = "tea/{}".format(tea_counter)
  datastore.hmset(tea_key,{"name": name,"brew-time": time,"box-size": size})
  return tea_key

```
Now, we'll import the Redis module for Python, instantiate a StrictRedis class,and create three teas as hashes from a Python shell with this function:

```
>>> import redis
>>>tea_datastore = redis.StrictRedis()
>>>earl_grey = create_tea(tea_datastore, "Earl Grey", 5, 15)
>>>earl_grey
'tea/1'
>>>tea_datastore.hgetall(earl_grey)
{b'box-size': b'15', b'name': b'Earl Grey', b'brew-time': b'5'}
>>>lavender_mint = create_tea(tea_datastore, "Lavender Mint", 2,20)
>>>peppermint_punch = create_tea(tea_datastore, "Peppermint Punch",4, 10)""""""
```
To add individual tea bags to the first box of tea for each of the tea types, we'll create a second function:

```
def add_box_of_tea(datastore, tea_key, number):
  box_counter = datastore.incr("global/{}/boxes".format(tea_key))
  tea_box_key = "{}/box/{}".format(tea_key, box_counter)
  datastore.sadd(tea_box_key, *range(1,number+1))
  return tea_box_key
```
We'll add the first box for each type of tea from our Python shell, as shown here:
```
>>>earl_grey_box_1 = add_box_of_tea(tea_datastore, earl_grey, 15)
>>> earl_grey_box_1
'tea/1/box/1'
>>>tea_datastore.smembers(earl_grey_box_1)
{b'2', b'1', b'3', b'5', b'4', b'13', b'10', b'11', b'14', b'7',
b'12', b'15', b'6', b'8', b'9'}
>>>lavender_mint_box_1 = add_box_of_tea(tea_datastore,lavender_mint, 15)
>>>tea_datastore.scard(lavender_mint_box_1)
15
>>>peppermint_punch_box_1 =add_box_of_tea(tea_datastore,peppermint_punch, 10)
>>>tea_datastore.scard(peppermint_punch_box_1)
10
```

Now, we will add a third function that takes the Redis instance and tea box key,brews a random tea bag from each box by setting an expiration time equal to the number of seconds of brew time for that type of tea, adds the tea bag key to a
brewing set, and finally returns the tea_bag_key :

```
def start_brew(datastore, tea_box_key):
    tea_box = tea_box_key.split("/box")[0]
    # Brew time is in minutes, we multiple by 60 for expire in seconds
    expire_time = int(datastore.hget(tea_box, "brew-time"))*60
    tea_bag_number = datastore.spop(tea_box_key)
    tea_bag_key = "{}/bag/{}".format(tea_box_key,tea_bag_number.decode())
    datastore.set(tea_bag_key, "brew")
    datastore.expire(tea_bag_key, expire_time)
    datastore.sadd("brewing", tea_bag_key)
    return tea_bag_key 
```

Calling the start_brew function three times, once for each type of tea, creates three tea bag keys and sets the expiration time depending on the type of tea:

Our final Python function iterates through all of the tea bags in the brewing set, polls the remaining time for each tea bag with the TTL command, and then either prints a message with the remaining time until the tea is finished brewing or a message that the tea is ready to drink depending on whether there is any remaining time before the tea bag key expires in our tea datastore:

```
def poll_brewing(datastore):
    active_tea_bags = datastore.smembers("brewing")
    for tea_bag in active_tea_bags:
        time_left = datastore.ttl(tea_bag)
        if time_left> 0:
            print("{} seconds left for {}".format(time_left,tea_bag))
        else:
            print("{} Ready to Drink!".format(tea_bag))
            # Remove expired tea bag from brewing set
            datastore.srem("brewing", tea_bag)
```
We will call the poll_brewing Python function three times, once near the beginning:
```
>>>poll_brewing(tea_datastore)
80 seconds left for b'tea/2/box/1/bag/5'
215 seconds left for b'tea/3/box/1/bag/3'
243 seconds left for b'tea/1/box/1/bag/5'
```
Again in approximately 60 seconds:
```
>>>poll_brewing(tea_datastore)
22 seconds left for b'tea/2/box/1/bag/5'
157 seconds left for b'tea/3/box/1/bag/3'
185 seconds left for b'tea/1/box/1/bag/5'
```
And finally at around 90 seconds:
```
>>>poll_brewing(tea_datastore)
b'tea/2/box/1/bag/5' Ready to Drink!
124 seconds left for b'tea/3/box/1/bag/3'
152 seconds left for b'tea/1/box/1/bag/5'
```
If the value is altered for a key that has a timeout, for example, using the APPEND command for a Redis string, the timeout still continues. From a Redis-cli session, we can replicate setting a string value to a tea bag with a timeout of 300 seconds:

```
127.0.0.1:6379> SET tea/1/box1/bag/8 brew
OK
127.0.0.1:6379> EXPIRE tea/1/box1/bag/8 300
(integer) 1
```

First we'll check to see what is the remaining TTL for tea/1/box/1/bag/8 , and then we will add an additional text to the value held at the key, and check the TTL again, as shown here:

```
127.0.0.1:6379> TTL tea/1/box1/bag/8
(integer) 288
127.0.0.1:6379> APPEND tea/1/box1/bag/8 ing
(integer) 7
127.0.0.1:6379> GET tea/1/box1/bag/8
"brewing"
127.0.0.1:6379> TTL tea/1/box1/bag/8
(integer) 259
```

If SET or GETSET is called on a key with a set timeout, the timeout will be cleared and then the key won't be evicted from the database. So, if SET is called on the tea/3/box/1/bag/3 key before it has expired, when the TTL is called on the tea/3/box/1/bag/3 key , Redis responds back with -1 . This is a default message for the keys that do not have a timeout.

```
127.0.0.1:6379> TTL tea/3/box1/bag/3
(integer) 225
127.0.0.1:6379> SET tea/3/box1/bag/3 brew
OK
127.0.0.1:6379> TTL tea/3/box1/bag/3
(integer) -1
```

So far, we have used the TTL as a polling mechanism to retrieve the value of any remaining timeouts that have been set on the keys we are interested in. Using client-side polling does have disadvantages, one of which is that if a delay occurs between the client-code and the server, our tea could become over-brewed. Redis offers a notification mechanism based on Pub/Sub that can be set up to send a message when a key expires, functionality that we'll explore in a later chapter on Redis messaging.

You can also use the PERSIST command to clear out a timeout that has been set on an existing key. Finally, calling EXPIRE on a key that has had a previous timeout set will clear out and set a new timeout.

```
127.0.0.1:6379> TTL tea/1/box1/bag/8
(integer) 118
127.0.0.1:6379> EXPIRE tea/1/box1/bag/8 300
(integer) 1
127.0.0.1:6379> TTL tea/1/box1/bag/8
(integer) 295
```
Even though this example is contrived, it should illustrate the basic operations of Redis key expiration. Redis sets the TTL with EXPIRE using an absolute UNIX timestamp from the underlying operating system. If you set a timeout on a key but
then shut down the Redis database with the data saved into a snapshot, restarting the Redis database after the timeout has expired will evict the key automatically.

Redis uses two methods for doing the actual expiration in the key-space, the first is if the key is actively requested by a client, and the second method is a probabilistic algorithm that randomly tests 20 keys with an associated expiration time stamp and deletes all of the keys in the sample that have expired.

In this chapter's next topic, we will take this knowledge about key expiration and see how Redis can modify it's behavior when it reaches its maximum allowed memory and keys within the key-space have timeouts set.

# LRU key evictions
page 58

reference: 
https://redis.io/topics/lru-cache

To demonstrate the various options for key evictions in Redis, we'll start with a simple example by setting a small memory Redis instance that the maxmemory directive sets to 1 megabyte. The maxmemory directive allows you set a hard upper
bound on the amount of memory that is available to a running Redis instance. 

Echoing the warnings in the default redis.conf file, setting the maxmemory has ramifications that we'll now see. To start with, we'll just create a very simple Redis key schema, that of generating and storing a unique id for a web application. After connecting to a Redis instance through Redis-cli, we'll run the following commands to clear out our datastore and then set the maxmemory directive to 1 megabyte:

```
127.0.0.1:6379> FLUSHALL
OK
127.0.0.1:6379>CONFIG SET maxmemory 1mb
OK
```
Next, we'll implement a function that takes a Redis instance, increments a global uuid , and then generates a random UUID from the standard uuid Python module. 
The add_id function code in Python is presented as follows in this code snippet:

```
>>> import uuid
>>>def add_id(redis_instance):
     redis_key = "uuid:{}".format(redis_instance.incr("global:uuid"))
     redis_instance.set(redis_key, uuid.uuid4())
```

When Redis runs out of memory, the default behavior - the noeviction policy - is illustrated in the following image:

The default maxmemory-policy policy is noeviction. In noeviction, no keys are set to expire and any write commands will cause a Redis error if there is no available memory to Redis. To confirm that our directives are set for experiment, we will
first check whether the maxmemory and maxmemory-policy are set to 1 megabyte and a noeviction policy respectively that is confirmed by checking the values of this directives by running the following CONFIG GET commands in our Redis-cli session:
```
127.0.0.1:6379>CONFIG GET maxmemory
1) "maxmemory"
2) "1048576"
127.0.0.1:6379>CONFIG GET maxmemory-policy
1) "maxmemory-policy"
2) "noeviction"
```

To test the noeviction policy, we will run a loop in Python until we receive an error, as shown here:

```
>>> while 1:
add_id(local_redis)
```
Quickly, we receive an exception from the Redis client, as follows: redis.
exceptions.ResponseError , OOM command not allowed when used memory
> 'maxmemory' .

Our while loop cycles through 181 times before we hit the 1 megabyte memory limit, with our counter value set at 181 . Now to check the state of our datastore by running the INFO memory command from the our Redis-cli and looking at the used_memory_peak_human :

```
127.0.0.1:6379> INFO memory
# Memory
used_memory:1048608
used_memory_human:1.00M
used_memory_rss:1769472
used_memory_peak:1048608
used_memory_peak_human:1.00M
used_memory_lua:35840
mem_fragmentation_ratio:1.69
mem_allocator:libc
```

Now, if we retrieve the global uuid counter key we set in the add_id function, we can see that we have 181 UUID stored in our 1 megabyte datastore which is the same value as our counter variable we incrementally increase by one in our loop:

```
127.0.0.1:6379> GET global:uuid"181"
127.0.0.1:6379> GET uuid:181
"1930a94e-38ff-4dbd-8885-eb44aed96122"
```
From the Redis-cli, we'll test the noeviction policy by trying to increment a second variable, like tmp:1 , this is the error we receive:

```
127.0.0.1:6379> INCR tmp:1
(error) OOM command not allowed when used memory > 'maxmemory'.
```

When Redis tries to execute any write ( SET , INCR , SADD , HSET , and so on) or other commands that increase memory usage under no available memory conditions, you will receive an error similar to the one seen in the preceding snippet. The next policy we'll examine is how Redis handles LRU when one or more keys have a timeout set.
The first expiration LRU policy named volatile-lru evicts the less recently used keys but only if those keys have an expiration timeout set with EXPIRE SET . If there are not any keys that are eligible for eviction, Redis will return the same exception when trying to write as in the noeviction policy.


An important note when using this policy is that when Redis runs out of memory it will start deleting keys that have an expiration timeout even if there is time remaining for the key. To test the volatile-lru policy, we'll flush our Redis instance. Running the same loop without setting an eviction time on any of the keys results in the same behavior as the noeviction policy:

```
127.0.0.1:6379>FLUSHDB
127.0.0.1:6379>CONFIG SET maxmemory-policy volatile-lru
127.0.0.1:6379> GET global:uuid
"181"
127.0.0.1:6379> INFO memory
# Memory
used_memory:1048608
used_memory_human:1.00M

```
Now, we're going to create a second function based upon the add_id and use a new add_id_expire function to set an expiration time of 300 seconds on the first 75 keys we create.

```
>>>def add_id_expire(redis_instance):
     count = redis_instance.incr("global:uuid")
     redis_key = "uuid:{}".format(count)
     redis_instance.set(redis_key, uuid.uuid4())
     if count <= 75:
        redis_instance.expire(redis_key, 300)
```
Resetting our counter variable to zero and running our test again, we iterate through the loop 238 times, 57 more times than when we ran our Redis instance with the noeviction policy. The loop results are confirmed when we retrieve our global increment variable global:uuid , check if uuid:1 and uuid:75 still exist, and check if uuid:76 and uuid:238 exist with uuids in our datastore:

```
127.0.0.1:6379> GET global:uuid
"238"
127.0.0.1:6379> GET uuid:1
(nil)
127.0.0.1:6379> GET uuid:75
(nil)
127.0.0.1:6379> GET uuid:76
"9922e314-17f8-4630-a709-07a3c8a8019c"
127.0.0.1:6379> GET uuid:238
"1a1318ae-57b6-4a4a-a366-2727033a315d"
```

As we expected, the keys uuid:1 to uuid:75 don't exist and during the loop an additional 57 keys were created in the database compared to the default noeviction policy. We can also see that we didn't create an additional 75 uuids only 57 more. 
This is likely due to the fact that we ran out of memory after all of the keys with expiration times had been evicted as we tried to add more keys without accounting for the memory overhead needed for eviction.

 
Now, the next LRU-style eviction policy, allkeys-lru , is recommended if you expect to add a power-law access pattern1 to your Redis database. The allkeys-lrupolicy is a good initial choice if you are unsure of what eviction policy to use with this important caveat. The allkeys-lru can delete any key in Redis and there is no way to restrict which keys are to be deleted. If your application needs to persist some Redis keys (say for configuration or reference look-up) don't use the allkeys-lru policy!

To test allkeys-lru , we'll flush the data from our Redis instance, set the maxmemory-policy directive, and then run our original add_id function.

Running our experiment in Python with an infinite while loop using the original add_id function, we went through hundreds of thousands of iterations (615,094) before running out of memory. Running an INFO stats command in the Redis cli and looking at the number of evicted keys in with the INFO stats , we see the following:

```
127.0.0.1:6379> INFO stats
# Stats
total_connections_received:2
total_commands_processed:1230193
.
.
evicted_keys:264524
```

In the allkeys-lru policy, Redis was able to process 1,230,193 commands by evicting 264,524. Since all of our keys in this loop have the same usage (that is, we are not retrieving any of the keys after our initial SET command), the Redis estimation of LRU for the keys was consistent across the datastore. For our experiment, the allkeys-lru eviction policy was effective in freeing memory for additional keys by evicting stale keys in our datastore. To further test the allkeys-lru policy, we will restart our experiment and only iterate 200 times through our loop (more than our test of the noeviction policy but less then our volatile-lru test ). The following are the results of this iteration:

```
127.0.0.1:6379> GET global:uuid
"200"
127.0.0.1:6379> INFO stats
# Stats
total_connections_received:2
total_commands_processed:614
.
.
evicted_keys:24
```

In this test, the allkeys-lru policy evicted 24 keys, we created the full 200 keys as we would expect going through 200 iterations of our loop, but the problem is that we don't know which of the 200 keys were evicted. To determine the missing keys, we'll need to loop through all of our keys and test if the key exists or has been evicted. 

This can easily be accomplished with the following Python code snippet:

```
>>> for i in range(1, 201):
    key = "uuid:{}".format(i)
    if not local_redis.exists(key):
      print(key)

```
Here is the output of this code snippet (your' results from running this code should vary from this list, if only slightly because of the probabilistic nature of Redis's LRU algorithm):

```
uuid:15
uuid:17
uuid:23
uuid:29
uuid:39
uuid:46
uuid:50
uuid:57
uuid:67
uuid:68
uuid:83
uuid:86
uuid:89
uuid:110
uuid:116
uuid:121
uuid:128
uuid:130
uuid:146
uuid:147
uuid:150
uuid:151
uuid:175
uuid:176
```
There are a couple of things to note about these evicted keys that illustrate the RedisLRU algorithm; first, the RedisLRU algorithm is not exact, as Redis does not automatically choose the best candidate key for eviction, the least used key, or the key with the earliest access date. Instead, Redis default behavior is take a sample of five keys and evict the least used of those five keys. Going back to the preceding list of evicted keys, we can see the results of this sampling strategy. If we want to increase the accuracy of the LRU algorithm, we can the change the maxmemory-samples directive in either redis.conf or during runtime with the CONFIG SET maxmemory-samples command. Increasing the sample size to 10 improves the
performance of the RedisLRU so that it approaches a true LRU algorithm but with the side-effect of more CPU computation.

Decreasing the sample size to 3 reduces the accuracy of RedisLRU but with a corresponding increase in processing speed.

The next two maximum memory eviction policies—volatile-random and allkeys-random —mirror the volatile-lru and allkeys-lru policies but do not use the LRU algorithm. The volatile-random policy evicts a random key based on expiration status that have been set on any keys. The entire keyspace is open for eviction in the allkeys-random policy.

We can use both the add_id and add_id_expire functions to test these two policies. 
First we'll run through the volatile-random policy with our experiment using a modified version of the add_id_expire function that sets half of the Keys to an expiration time of 5 minutes, allowing us to compare the performance of the volatile-random policy to our other policies. Our results are different to the volatile-lru test when we created a total of 246 keys. Unlike the volatile-lru policy, we need to an O(n) operation to figure if a uuid:1 through the last uuid key we created were evicted.

Running a sample run of 1000 iterations of the add_id_expire function results in the following performance:

```
127.0.0.1:6379> INFO stats
# Stats
total_connections_received:2
total_commands_processed:1805
.
.
expired_keys:0
evicted_keys:499
```

Notice that even though our add_id_expire function sets an expiration time on half of keys added to our Redis instance, the small size of our sample set was such that all of the keys were evicted before any of the keys expired under this test of the volatile-lru policy. Checking the state of our keyspace, we find the following:

```
127.0.0.1:6379> GET global:uuid
"652"
127.0.0.1:6379> GET uuid:1
(nil)
127.0.0.1:6379> DBSIZE
(integer) 153
127.0.0.1:6379> GET uuid:652
(nil)
127.0.0.1:6379> GET uuid:651
"e42ce917-efe9-4657-b2d6-cccd0f26b19c"
```
In this test of the volatile-random policy, we ran out of memory before we could complete the 1,000 iterations. During those 652 iterations – easily determined by our GET global:uuid call – of our test, we created and evicted 499 keys while
retaining only 153 keys.

The last maximum memory policy is volatile-ttl . It is similar to volatile-lru but with the additional characteristic that Redis will try to evict those keys based on the time to live (TTL) of the key that is to be evicted from the Redis database.

# Creating memory efficient Redis data structures
page 67
The following are some of the methods for memory optimization in Redis:

## Small aggregate hashes, lists, sets, and sorted sets
For hashes, lists, and sorted sets, this special encoding is based on ziplist A Ziplist is  described from ziplist.c as follows:

The ziplist is a specially encoded dually linked list that is designed to be very memory efficient. It stores both strings and integer values, where integers are encoded as actual integers instead of a series of characters. It allows push and pop
operations on either side of the list in O(1) time. However, because every operation requires a reallocation of the memory used by the ziplist, the actual complexity is related to the amount of memory used by the ziplist.2 .

Depending on the size, type, and contents of the data structure, the ziplist encoding offers significant memory savings for your Redis database. Redis dynamically switches between the ziplist and the default encoding for the data structure when
the current limit for that Redis data type is reached. To see this switch in action,we'll create a Python function that displays this dynamic switch by printing the type of encoding and size for a hash key when its default threshold is met:

```
def dynamic_encoding_switch(instance):
    for i in range(515):
        instance.hset("test-hash", i, 1)
        if i> 510:
            debug = instance.debug_object("test-hash")
            print("Count: {} Length: {} Encoding: {}".format(i,debug.get('serializedlength'), debug.get('encoding')))
```
Running dynamic_encoding_switch results in the following output to our Python shell:

```
>>>dynamic_encoding_switch(local_redis)
Count: 511 Length: 2070 Encoding: ziplist
Count: 512 Length: 2439 Encoding: hashtable
Count: 513 Length: 2444 Encoding: hashtable
Count: 514 Length: 2449 Encoding: hashtable
```
Why does Redis dynamically re-encode the hash from a ziplist to a hash table in this case? This is because there is a trade-off between memory efficiency and performance. The ziplist implementation in Redis achieves it's small memory size by storing only three pieces of data per entry; the first is the length to the previous entry, the second is the length of the current entry, and the third is the stored data. 

This brevity comes at the cost of more computation (hence time) that is required for changing the size and retrieving the entry versus the larger linked-list based encodings that store additional pointers but is correspondingly faster in changing
and retrieving at larger sizes.

For hashes, the hash-max-ziplist-entries directive sets the total number of fields that can be specially encoded as a ziplist with a default value of 512 fields. The hash-max-ziplist-value directive sets the maximum size before the hash is converted from a ziplist with a default size of 64. We can illustrate these two conditions with the following very simplistic examples.

To test the size difference between ziplist and linked list for hashes, let's spin up two identical instances of Redis. For our remaining tests in this chapter, we will keep  our first instance's configuration directives using Redis's default values and modify the second instance's directives to force Redis to use the default encoding for each
data type.

First, we'll set the hash-max-ziplist-entries to 0 for the second instance, create and populate a hash with 500 fields containing identical fields and integers values then compare the two using the DEBUG OBJECT command from the Redis-cli. for
these two Redis hashes. First, we'll create a small Python function to create our hash:

```
def plot_hashes(runs=500):
    reset()
    key = "test-hash"
    INSTANCE2.config_set('hash-max-ziplist-entries', 0)
    run, zip_list, linked_list = [], [], []
    for i in range(runs):
      field = "f{}".format(i)
    INSTANCE1.hset(key, field, i)
    INSTANCE2.hset(key, field, i)
    debug1 = INSTANCE1.debug_object(key)
    zip_list.append(debug1.get("serializedlength"))
    debug2 = INSTANCE2.debug_object(key)
    linked_list.append(debug2.get("serializedlength"))
```

Now, we'll run our function and then examine the results of the debug_object command on test-hash for each object, as shown here:

```
>>>plot_hashes()
>>INSTANCE1.debug_object("test-hash").get("serializedlength")
3102
>>>INSTANCE2.debug_object("test-hash").get("serializedlength")
3764
```
If you compare the encoding size for these two identical hashes, you will find that the standard hashtable encoding for test-hash results in a serialized length of 3764 while the ziplist encoding for test-hash is 3102, a direct memory saving
(when serialized) of 662 bytes. Graphing the results as presented in the following figure gives you only the memory savings but does not consider the additional computation time the ziplist encoding requires as the size of hash increases:

page.69

or lists, like hashes, the ziplist encoding is used for small lists with the thresholds being determined by the list-max-ziplist-entries and the list-max-ziplist-value with both directives having the same default values as the hash directives of
512 and 64 respectively. In the code file small_types_tests.py , there is a function that sets the list-max-ziplist-value to 0 for the second instance. The function then iterates through a number of runs, adding a random UUID to the same Redis
key in each Redis instance, and then saving the serialized length of each instance as seen in this Python code snippet from the small_types_tests.py :
```
def plot_list_ziplist(runs=1000):
    reset()
    INSTANCE2.config_set("list-max-ziplist-entries", 0)
    key = "test-list"
    run, zip_list, linked_list = [], [], []
    for i in range(runs):
        run.append(i)
        uid = uuid.uuid4()
        INSTANCE1.lpush(key, uid)
        INSTANCE2.lpush(key, uid)
        debug1 = INSTANCE1.debug_object(key)
        zip_list.append(debug1.get("serializedlength"))
        debug2 = INSTANCE2.debug_object(key)
        linked_list.append(debug2.get("serializedlength"))
```
We can see this difference between by comparing the two list encoding methods on test-list in both Redis instances:

```
>>>INSTANCE1.debug_object("test-list").get("serializedlength")
15756
>>>INSTANCE2.debug_object("test-list").get("serializedlength")
18946
```
For the first 512 UUIDs that are added to both lists, the size of the ziplist encoded value is 15,787 while the size of the linked-list encoded value is 18,946, a memory savings of 3190 bytes if using a ziplist.

While it may not be immediately apparent, but for very small lists, the linked-list encoding for Redis lists is actually more efficient. We can see this more clearly if we graph the first 50 list items as shown here:

page 71
For small lists, the default linked list encoding is more efficient than the ziplist encoding but as the size of the list increases, the ziplist encoding becomes more efficient.

For sets the advantages of these special encodings only occurs if the set is small and contains only integers. The Redis directive, set-max-intset-entries with a default value of 512 will encode the set as an intset data type. Running the two
Redis instances experiment with the Python function and then retrieving the output results in:

```
>>>INSTANCE1.debug_object("test-set").get("serializedlength")
1034
>>>INSTANCE2.debug_object("test-set").get("serializedlength")
2874
```

Finally, in examining and experimenting with the special encoding of sorted sets, we will use the same two Redis instances and set the Rediszset-max-ziplist-entries to 0 to force Redis to use a hashtable for the data encoding. For each ZADD
command, the plot_sortedset function adds a UUID to each instance with the score set to 0 for lexical ordering of the UUIDs. To Author: Unclear? Please rephrase for more clarity.

Comparing the results from our Python shell on the test-sorted-set in both instances results in the following:

```
>>>INSTANCE1.debug_object("test-sorted-set").get("serializedlength")
4421
>>>INSTANCE2.debug_object("test-sorted-set").get("serializedlength")
4994
```

For the ziplist implementation, the test-sorted-set serialized length is 4,421 bytes while the skiplist implementation serialized length of 4,994 a difference in favor of the ziplist encoding of 573 bytes.

For all the ziplist implementations, the computation time increases significantly as the size of the data structure grows larger. For your own Redis-based application, adjusting these thresholds is a matter of balancing memory size with performance realizing that any large ziplist data structure slows down when compared to equivalent default data structure for the Redis data type.

# Bits, bytes, and Redis strings as random access arrays 
page 74

In both chapters one and two, we talked about the use of bitmaps in Redis using various commands such as SETBIT , GETBIT , BITCOUNT , BITPOS , and BITOP .
To illustrate the memory savings of using bitmap over a set, let's return to the tea example from earlier and look at how we could indicate whether a tea was decaffeinated or not, an important decision for those of us trying to wake up or fall
asleep! A Redis set, with a key name of tea:caffeinated could easily solve this issue. We could store the key name of each tea with caffeine in the set with the SADD command. For the sake of this example, let's assume that we have over 10,000 teas
and over 60% has some traceable level of caffeine. Continuing with our initial Redis key schema for teas and since we are using a unique incremental counter for each tea, the tea/caffeinated set could be populated from a Redis-cli like this (assuming tea:4 is something like green tea):

```
127.0.0.1:6379> SADD teas/caffeinated tea:1 tea:4
(integer) 2
```
To simulate our full 10,000 tea inventory, we'll use a function in the small_types_tests.py Python module, populate_tea , to compare our initial approaches:

```
def populate_tea(full=True):
    for i in range(10000):
        if random.random() <= .6:
            member = i
            if full:
                member = "tea/{}".format(i)
                INSTANCE1.sadd("teas/caffeinated", member)
                INSTANCE1.setbit("teas/caffeine", i, 1)
```
In populate_tea , we call the random.random() method to generate a random value between 0 and 1 . To model our assumption that 60% of our teas have some caffeine,we check to see if the value is below .6 , and add a tea with caffeine to both our set, teas:caffeinated and our bitmap teas:caffeine . Running this function we have over 10,000 variety of teas:

```
>>>INSTANCE1.debug_object("teas/caffeinated").get("serializedlength")
53879
>>>INSTANCE1.debug_object("teas/caffeine").get("serializedlength")
1252
```

To see how many teas in our inventory have caffeine, we can either run the SCARD command on the teas/caffeinated set or the BITCOUNT command on the teas/caffeine bitmap:

```
127.0.0.1:6379> SCARD teas/caffeinated
(integer) 6063
127.0.0.1:6379> BITCOUNT teas/caffeine
(integer) 6063
```

For our teas/caffeinated set, the serialized length is 53,879 bytes while our bitmap is stored as a raw string with a serialized length of 1,252 a significant memory savings of 52,627 bytes! Now, you might be wondering why we have the full parameter in our populate_tea function? With our initial implementation, we stored a string key for each tea. To reduce the size of the teas:caffeinated set, we set the function parameter full to False and then we only store the integer of the tea counter. So, running populate_tea with full=False, results in the following lengths for both teas/caffeinated and the teas/caffeine from the Redis-cli:

```
>>>INSTANCE1.debug_object("teas/caffeinated").get("serializedlength")
17800
```
By using integers instead of strings in our teas/caffeinated set, we were able to drop the size significantly from 52,879 to 17,800 bytes but our bitmap teas/caffeine is still orders of magnitude smaller than our set. However, using bitmaps is not necessarily a panacea. Sparse bitmaps that number in the hundreds of millions waste space as only the first bit is set per offset, leaving the remaining bits per entry not being used.
Also, Redis integer sets and hashes provide additional functions that are difficult or require a lot of client code to work correctly with bitmaps.

# Optimizing hashes for efficient storage
page 75
In the Redis topic on Memory Optimization, Salvatore Sanfilippo writes about a technique using Redis's hashes to implement a high-level and very memory-efficient key-value data storage using Redis hashes. To illustrate how to use this technique,
we will return to the legacy representation of MARC records introduced in Chapter 1, Why Redis? and compare two approaches to storing MARC JSON serialization in Redis. In the first experiment we will simply use a one-to-one MARC redis key to MARC record JSON serialization stored as a string. For the second experiment,we will use hashes to store the same JSON serializations.
