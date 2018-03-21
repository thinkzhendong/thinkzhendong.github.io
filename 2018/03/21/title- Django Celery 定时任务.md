---
title: Django Celery 定时任务
---
环境信息：
1. python 3.6.2
2. Django 1.9.8
3. celery 4.1.0
4. django-celery-beat 1.1.1

# 准备工作 安装环境
安装 django-celery-beat 及进行配置
    1. pip install -U django-celery-beat
    如果项目中settings.py中设置了USE_TZ=False，那么建议使用源码安装，有一个bug需要修改源文件     的方式进行解决。随后在项目根目录，进行migrate：
    2. python manage.py migrate
    3. 配置 项目/项目配置文件夹/settings.py:
        ...

    INSTALLED_APPS = (
       	...,
       	'django_celery_beat',
    )
    
    ...
    CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers.DatabaseScheduler'
    
Redis安装及启动：
 1. 安装redis：
    Mac上通过brew 进行安装
 2. 列表项

    输入命令: brew install redis

2. 启动redis：
    sudo /usr/local/bin/redis-server
3. 查看redis进程及端口：
    ps aux|grep redis

celery 安装
    pip install -U celery[redis]

# Django 集成celery：
- 项目名称/
  - manage.py
- 项目配置文件/
    - __init__.py
    - settings.py
    - urls.py
    - celery.py
- mails/  # 这是一个app
    - models.py
    - views.py
    - tasks.py
# 配置以下文件 
1. 项目下的celery.py
    from __future__ import absolute_import, unicode_literals
    import os
    import django
    from celery import Celery
    import datetime
    
    os.environ.setdefault('DJANGO_SETTINGS_MODULE','项目.settings')
    django.setup()  # 如果 tasks 里边涉及对 model 的操作，则需要加上     django.setup()
    app = Celery('项目名')
    
    app.now = datetime.datetime.now  # 不加这一行会报错
    app.config_from_object('django.conf:settings', namespace='CELERY')
    app.autodiscover_tasks()  # 自动加载各个app下边的tasks.py

2. 配置__init__ 路径:项目/配置文件夹/\__init__.py:
    from __future__ import absolute_import, unicode_literals
    from .celery import app as celery_app
    \_\_all__ = ['celery_app']

3. 配置settings.py
    CELERY_BROKER_URL = 'redis://127.0.0.1:6379/0'  # 这里注意redis的端口

4. 在tasks文件写入自己的使用逻辑
            from 定义的项目名称.celery import app
                @app.task
                def ceshi_celery():
                    Ceshi.objects.create(type=1, text='测试plan底下文件是否运行2')
                另:
                    也可以使用 from celery import shared_task
                            @shared_task
                以上为使用例子
5. 
    将django-celery-beat新增的model，配置到后台：
    看django_celery_beat.model可以看到一共是有5个model，具体的功能可以看源码，也很容易懂，这里主要用的是IntervalSchedule、CrontabSchedule和PeriodicTask。因为这里用的是xadmin后台的框架，但后台框架不会影响定时任务的功能，就不赘述。
需要注意Task name字段，为任务的路径或者名字，如果按照上述步骤配置的话，Task name应为mails.tasks.do_something。如果使用了@shared_task(name='new_task_name')进行重命名，那么Task name应为new_task_name
另外，如果是带参的定时任务，可以通过Arguments、Keyword arguments进行传参。

# 接下来启动celery 在项目下运行:
    celery -A 定义的项目名称 worker -l info
    celery -A 定义的项目名称 beat -l info

参考：
    https://actmerce.github.io/2018/03/21/django-celery-beat-1/