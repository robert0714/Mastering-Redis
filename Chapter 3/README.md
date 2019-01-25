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
