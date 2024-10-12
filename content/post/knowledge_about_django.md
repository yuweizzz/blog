---
date: 2022-01-15 23:22:45
title: Django 知识笔记
tags:
  - "Python"
  - "Django"
draft: false
---

这篇笔记用来记录一些 Django 常用的项目组件。

<!--more-->

```bash

                                       (@@) (  ) (@)  ( )  @@    ()    @     O     @     O      @
                                  (   )
                              (@@@@)
                           (    )

                         (@@@)
                       ====        ________                ___________
                   _D _|  |_______/        \__I_I_____===__|_________|
                    |(_)---  |   H\________/ |   |        =|___ ___|      _________________
                    /     |  |   H  |  |     |   |         ||_| |_||     _|                \_____A
                   |      |  |   H  |__--------------------| [___] |   =|                        |
                   | ________|___H__/__|_____/[][]~\_______|       |   -|                        |
                   |/ |   |-----------I_____I [][] []  D   |=======|____|________________________|_
                 __/ =| o |=-~O=====O=====O=====O\ ____Y___________|__|__________________________|_
                  |/-=|___|=    ||    ||    ||    |_____/~\___/          |_D__D__D_|  |_D__D__D_|
                   \_/      \__/  \__/  \__/  \__/      \_/               \_/   \_/    \_/   \_/

```

## 创建 Django 项目

在虚拟环境中创建 Django 项目。

```bash
# 激活虚拟环境
$ source project/bin/activate

# 安装 Django 3.2
(project) $ pip install django

# 初始化 Django 项目
# django-admin startproject name [directory]
(project) $ django-admin startproject new_project

# 在项目中添加 app
(project) $ django-admin startapp new_app

# 测试项目
(project) $ python manage.py runserver
```

不带 directory 的初始化会在当前工作目录创建新的项目目录，然后你可以在其中找到 `manage.py` 和保存项目配置文件的目录。

## 使用 MySQL 做后端存储

Django 默认的后端存储是 sqlite3 ，可以用更强大的 MySQL 或者 PostgreSQL 代替。

```bash
# 安装依赖
(project) $ pip install mysqlclient

# 安装后需要将以下内容写入到 settings.py 中
(project) $ cat settings.py
...
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'database_name',
        'HOST': '127.0.0.1',
        'PORT': '3306',
        'USER': 'root',
        'PASSWORD': 'mysecret',
        'ATOMIC_REQUESTS': True,
    }
}
...

# Django 不会自动生成 database ，需要手动创建数据库
mysql > CREATE DATABASE database_name
     -> DEFAULT CHARACTER SET utf8  # 尽量把字符集设置为 utf8 ，这样可以将中文字符存入数据
     -> DEFAULT COLLATE utf8_general_ci;

# 在 app 中定义 model 后，需要执行迁移命令生成对应的表
(project) $ python manage.py makemigrations
(project) $ python manage.py migrate
```

如果因为已有数据库的字符集不是 utf8 而导致的中文字符乱码，可以这样做：

```bash
# 查看表的详细信息
mysql > SHOW CREATE TABLE table_name;

# 修改 database 的字符集，但这对已有字段不起效果
mysql > ALTER DATABASE db_name DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;

# 修改所有字段的字符集，可以即时生效
mysql > ALTER TABLE table_name CONVERT TO CHARACTER SET utf8 COLLATE utf8_general_ci;
```

## 使用 Redis 做缓存引擎

低版本的 Django 原生支持 Memcached 作为缓存引擎，我们可以额外安装扩展来支持 Redis ，而 Django 4.0 已经原生支持 Redis 作为缓存引擎。

可选的 Redis 扩展有很多，比较推荐使用 django-redis 和 python-redis-lock 的组合应用。

```bash
# 只使用 django-redis
(project) $ pip install django-redis

# 安装后需要将以下内容写入到 settings.py 中
(project) $ cat settings.py
...
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
            "PASSWORD": "mysecret"
        }
    }
}
...

# 使用 django-redis 和 python-redis-lock 的组合
(project) $ pip install "python-redis-lock[django]"
# 这是 python-redis-lock 推荐使用的方法，所有扩展都会自动安装

# 安装后需要将以下内容写入到 settings.py 中
(project) $ cat settings.py
...
CACHES = {
    'default': {
        'BACKEND': 'redis_lock.django_cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient'
        }
    }
}
...

# 原生的 django-redis 没有 lock 函数，使用 python-redis-lock 可以较好应对并发的问题
(project) $ cat cache_test.py
from django.core.cache import cache

def function():
    val = cache.get(key)
    if not val:
        with cache.lock(key):
            val = cache.get(key)
            if not val:
                # DO EXPENSIVE WORK
                val = ...
                cache.set(key, value)
    return val
```

## 使用 Celery 做任务队列

Celery 是分布式任务队列，可以非常方便地实现任务调度。

Celery 通过消息机制进行通信，生产者将消息发布到 Broker 中， Worker 会消费 Broker 中的消息，执行具体的任务内容， Celery 可以有多个 Worker 和 Broker ，实现高可用和横向扩展。

在已有的 Djnago 项目中使用 Celery 只需要简单添加修改几个文件即可。

```python
# 在 Django 项目中配置 Celery

# 假设已有项目名为 project ，那么应该在 project 配置文件的目录中新增这个文件
# project/project/celery.py
import os
from celery import Celery

# Set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'project.settings')

# Enable running a worker with superuser privileges, don't use it in production.
os.environ['C_FORCE_ROOT'] = 'True'

app = Celery('project')

# Using a string here means the worker doesn't have to serialize
# the configuration object to child processes.
# - namespace='CELERY' means all celery-related configuration keys
#   should have a `CELERY_` prefix.
app.config_from_object('django.conf:settings', namespace='CELERY')

# Set timezone
app.conf.timezone = 'UTC'

# Load task modules from all registered Django apps.
app.autodiscover_tasks()


# 在 Django 项目启动时，同时加载完成配置的 Celery 实例
# project/project/__init__.py
# This will make sure the app is always imported when
# Django starts so that shared_task will use this app.
from .celery import app as celery_app

__all__ = ('celery_app',)


# 使用指定 namespace 的 config_from_object 从 Django settings 加载 Celery 配置
# 这里使用 Redis 作为 Broker 和任务执行结果的保存后端，实际可以根据情况使用不同的后端
# part of project/project/settings.py
CELERY_BROKER_URL = 'redis://127.0.0.1:6379/2'
CELERY_RESULT_BACKEND = 'redis://127.0.0.1:6379/3'
CELERY_TASK_SERIALIZER = 'pickle'
CELERY_RESULT_SERIALIZER = 'pickle'
CELERY_ACCEPT_CONTENT = ['json', 'pickle']


# 在具体 Django app 中添加任务实体，它会由 autodiscover_tasks 自动发现
# project/appA/tasks.py
from celery import shared_task

@shared_task
def add(x, y):
    return x + y
```

可以看到大概的配置步骤如下：

1. 创建 Celery 实例。
2. 在设置 Django 项目的环境变量完成后，从它们之中获取 Celery 实例需要的配置。
3. 执行自动任务发现，获取到项目中的定义完成的具体任务。

除了必要的 Broker ，你还需要手动启动 Worker 。

```bash
# 启动输出 INFO 级别日志的 worker
(project) $ celery -A project worker -l INFO
```

### Celery 的定时任务

Celery 提供了 beat 来实现任务的定时调度，我们可以将 shared_task 注册到 Celery 实例的 beat_schedule 中，它就会作为定时任务被自动调度执行。

```bash
$ cat project/project/celery.py
...
# Register period task
app.conf.beat_schedule = {
    'add-every-30-seconds': {
        'task': 'appA.tasks.add',
        'schedule': 30.0,
        'args': (1, 2),
    },
}
...
# 实际的 schedule 除了简单的定时时长，还可以使用 celery 提供的 crontab 和 solar 调度器来实现不同的调度周期
```

使用 beat 会产生 celerybeat-schedule 文件，它会存储任务的最后运行时间，我们需要在启动 Worker 的同时额外启动 beat 进程。

```bash
# 启动 beat 进程
(project) $ celery -A project beat

# 可以在启动 worker 时将 beat 嵌入 worker ，应用于测试场景
(project) $ celery -A project worker -B -l INFO

# -s 可以指定 celerybeat-schedule 的写入位置
(project) $ celery -A project beat -s /var/run/celerybeat-schedule
```

### 定义 Celery 工作流

Celery 提供了一些任务编排的基本函数，我们可以通过这些函数定义工作流。

首先需要先对具体的 shared_task 任务函数进行签名，然后对签名使用编排函数构建工作流，比较常用的有 chain 任务链， group 并行任务组和 chord 回调。

```python
# project/appA/tasks.py
from celery import shared_task

@shared_task
def add(x, y):
    return x + y

# 基本的函数签名
from celery import signature
signature_instance = signature('add', args=(2, 2))
# 函数签名简写
signature_instance = add.s(2, 2)

# group 用来并行执行任务，它的参数是一组函数签名组成的列表
signature_list = [add.s(1, 2), add.s(3, 4), add.s(5, 6)]
group_instance = group(signature_list)
# 直接调用 group 实例就会调用整个任务组
group_instance()

# chain 用来描述任务链，上一次任务结果会以参数形式传递到下一任务中
chain_instance = (add.s(1, 2) | add.s(3) | add.s(4))
# 直接调用 chain 实例会自动调度任务，如果中间有失败任务，整个工作链会中断
chain_instance()

# chord 用来执行回调，用于并行任务组完成后执行额外的任务
chord_instance = (group_instance | add.s())
# 直接调用 chord 的实际结果类似于 chain
chord_instance()
```

### Celery 的任务监控

使用 flower 可以对 Celery 队列运行情况提供 web 界面监控面板。

```bash
# 安装 flower
(project) $ pip install flower

# 需要先启动 Celery Workers
(project) $ celery -A project worker -l INFO

# 直接启动 flower
(project) $ flower -A project --address=127.0.0.1 --port=5555

# 通过 Celery 启动 flower
(project) $ celery -A project flower --address=127.0.0.1 --port=5555
```

## 使用 Django 序列化器

序列化实际上是比较广泛的定义，在 Django 中一般指将 Django Model 转换为通用型数据格式，最常用的就是 json 格式。

原生的 Django serializers 可以将 Model 转换为 json 格式的纯文本，除了 json 还包括 XML 和 YAML 这两种格式。

通常情况下会使用 Django REST framework 提供的 serializers 来替代原生的序列化器， Django REST framework 是非常强大的 Django 扩展库，在 Django 原有的基础类上做了增强封装，它的 serializers 拥有更丰富的内置函数，可以轻松实现序列化和反序列化。

```python
from django.core import serializers
from rest_framework import serializers as drf_serializers
from apps.models import DBModel
import json

# django serializers
string = serializers.serialize("json", DBModel.objects.all(), fields=('id', 'data'))
pyobjs = json.loads(string)

# DRF serializers
class DBSerializer(drf_serializers.ModelSerializer):
    class Meta:
        model = DBModel
        fields = ['id', 'data']

serializer = DBSerializer(DBModel.objects.all(), many=True)
pyobjs = serializer.data
```

反序列化则是将原生的数据格式转换为 Model Object ，是序列化的相反过程，这个过程使用 Django REST framework serializers 比使用原生 Django serializers 更加简单直接，并且支持数据合法性验证。

```python
from django.core import serializers
from rest_framework import serializers as drf_serializers
from apps.models import DBModel
import json

# django serializers
string = serializers.serialize("json", DBModel.objects.all(), fields=('id', 'data'))
for deserialized_object in serializers.deserialize("json", string):
    deserialized_object.save()

# DRF serializers
class DBModelSerializer(drf_serializers.Serializer):
    id = serializers.IntegerField(required=True)
    data = serializers.CharField(required=True, allow_blank=False, max_length=100)

    def create(self, validated_data):
        return DBModel.objects.create(**validated_data)

serializer = DBModelSerializer(data=json.loads(string))
serializer.is_valid()
serializer.save()
```

## 使用 Django ManyToMany Field

在 Django Model 支持的 field 中有一个特殊的类型 `ManyToManyField` ，用来实现 Model 之间的多对多关系，它会在数据库中新建一张表，这张表会保存两个 Model 之间的主键关系，从而可以通过某个 Model 的主键找到所有和它关联的另一种 Model 。

可以通过 Model 直接 `add` 或者 `remove` 来修改关联关系，但是也可以通过直接操作这张关系表来修改 Model 之间的关系。

```python
from django.db import models

class Publication(models.Model):
    title = models.CharField(max_length=30)

    class Meta:
        ordering = ['title']

    def __str__(self):
        return self.title

class Article(models.Model):
    headline = models.CharField(max_length=100)
    publications = models.ManyToManyField(Publication)

    class Meta:
        ordering = ['headline']

    def __str__(self):
        return self.headline

m2m_model = Article.publications.through
# 查询所有的 Model 关系
relation = m2m_model.objects.all()
```

## 使用 Django Signal

Django 内置支持 Signal ，原生支持的 Signal 主要有 Model 变动相关的信号和请求相关的信号，同时支持自定义信号。

```python
from django.dispatch import receiver
# 自定义信号
signal = django.dispatch.Signal()

# 发送信号
signal.send(sender=None, args_1='value_1', args_2='value_2')

# 接收器函数
@receiver(signal)
def signal_callback(sender, **kwargs):
    print(kwargs['args_1'], kwargs['args_2'])
```

其中 `send` 函数是信号的内置函数，通常使用 `sender` 来传递触发信号发送的对象，并使用它的一些自定义方法，但这个对象也是可以为空的。

内置的信号通过搭配 `partial` 使用，这样在参数传递时会更加简洁。

```python
from functools import partial
from django.db.models.signals import m2m_changed
from django.db import models

class Publication(models.Model):
    title = models.CharField(max_length=30)

    class Meta:
        ordering = ['title']

    def __str__(self):
        return self.title

class Article(models.Model):
    headline = models.CharField(max_length=100)
    publications = models.ManyToManyField(Publication)

    class Meta:
        ordering = ['headline']

    def __str__(self):
        return self.headline

# 模拟 Publication 对象被删除时发送的信号
m2m_model = Article.publications.through
send = partial(
    m2m_changed.send,
    sender=m2m_model,
    # Publication 变动而受到影响的 Article 实例
    instance=Article.objects.filter(...),
    # Article 包含了 Publication ，二者非反向关系
    reverse=False,
    # Publication 变动
    model=Publication,
    # Publication 变动实例的主键集
    pk_set=[ for i.id in Publication.objects.filter(...) ]
)
# 发送 pre_remove 信号
send(action="pre_remove")
```
