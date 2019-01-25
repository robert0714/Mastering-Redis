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
datastore.hmset(tea_key,
     {"name": name,
     "brew-time": time,
     "box-size": size})
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
defstart_brew(datastore, tea_box_key):
    tea_box = tea_box_key.split("/box")[0]
    # Brew time is in minutes, we multiple by 60 for expire in seconds
    expire_time = int(datastore.hget(tea_box, "brew-time"))*60
    tea_bag_number = datastore.spop(tea_box_key)
    tea_bag_key = "{}/bag/{}".format(tea_box_key,
        tea_bag_number.decode())
    datastore.set(tea_bag_key, "brew")
    datastore.expire(tea_bag_key, expire_time)
    datastore.sadd("brewing", tea_bag_key)
    return tea_bag_key""""""
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
