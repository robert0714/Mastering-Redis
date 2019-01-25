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
