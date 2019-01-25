# 32-bit Redis
```
apt install python-pip
pip install pymarc
pip install redis
apt-get install python-matplotlib


>>> import random
>>> import redis
>>> import uuid
>>> INSTANCE64 = redis.StrictRedis()
>>> INSTANCE32 = redis.StrictRedis(port=6380)

>>>def test_redis_32k_64k():
    for i in range(100000):
      key = "uuid:{}".format(i)
      value = uuid.uuid4()
      INSTANCE32.set(key, value)
      INSTANCE64.set(key, value)

>>> test_redis_32k_64k()
```
