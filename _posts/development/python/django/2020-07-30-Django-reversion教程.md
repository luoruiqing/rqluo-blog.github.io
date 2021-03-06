---
title: "Django 数据的版本控制"
subtitle: "Django | django-reversion | 版本控制 | Model | 历史记录 | 回滚 | 删除 | 恢复"
layout: post
author: "luoruiqing"
catalog: true
header-style: text
tags:
  - Django
---



在工作中经常会有需求 **保留操作历史**, 有时还需要能够回滚到某个版本, 甚至删除了也能恢复, django-reversion 能快速实现版本控制.

# [django-reversion](https://django-reversion.readthedocs.io/)

> django-reversion is an extension to the Django web framework that provides version control for model instances.


## 安装过程

- 安装模块 `pip install django-reversion`.
- 添加 `'reversion'` 到 `settings.py` 配置文件中 `INSTALLED_APPS` 的选项内
- 运行迁移 `manage.py migrate`.


> MySQL需要使用 **InnoDB** 引擎

## 快速开始

#### 模型注册

```python
from django.db import models
import reversion

@reversion.register() # 注册Model
class YourModel(models.Model):

    pass
```

#### Admin注册

`admin`注册后, 所有的改动将自动生成历史, 并可以回滚

```python
from django.contrib import admin
from reversion.admin import VersionAdmin

@admin.register(YourModel) # 注册表
class YourModelAdmin(VersionAdmin): # 修改继承VersionAdmin

    pass
```

#### 初始化

```sh
python manage.py createinitialrevisions # 初始化历史版本
```

> **注意: 每当注册新的`Model`时, 运行一次`createinitialrevisions`**


#### 查看结果


打开**已注册**的Model, 此时出现**删除恢复**按钮, 通过admin`删除`的模型即可通过此处`恢复`

![img]({{ site.baseurl }}/img/in-post/development/python/django/django-reversion/p1.png "...")

打开**已注册**的Model, 出现**历史**按钮, 通过admin`修改`的模型即可通过此处`回滚`

![img]({{ site.baseurl }}/img/in-post/development/python/django/django-reversion/p2.png "...")


**通过上述方式完成了`django-Admin`的恢复以及回滚, 正常操作Admin即可体验效果**


## 基础操作

#### 创建历史

`reversion.create_revision`

```python
# 开始创建历史版本
with reversion.create_revision():

    obj = YourModel(name="obj v1") # 这里是你的Model, 必须是注册过的Model
    obj.save() # 这里创建了新的YourModel实例

    # Store some meta-information.
    reversion.set_user(request.user) # 可以设置这个版本是由哪个用户创建的
    reversion.set_comment("Created revision 1")  # 设置版本的标注
    # 在上下文管理退出时, 自动将obj的第一个版本保存

# 对同一个对象创建第二个版本
with reversion.create_revision():
    obj.name = "obj v2"
    obj.save() # 对象变更

    # Store some meta-information.
    reversion.set_user(request.user)
    reversion.set_comment("Created revision 2")
```

上面的代码中是对一个`obj`保存了两个版本的信息, `with` 两块

#### 查看历史

```py
from reversion.models import Version

# instance 只的是注册过的Model实例对象, 例如 obj = YourModel.objects.get(id=1) 中, obj即为实例
versions = Version.objects.get_for_object(instance)
assert len(versions) == 2

# Check the serialized data for the first version.
assert versions[1].field_dict["name"] == "obj v1"

# Check the serialized data for the second version.
assert versions[0].field_dict["name"] == "obj v2"
```

`versions`中存放着一个实例的多个历史版本, 最新的下标为`0`

#### 回滚版本

```py
# 回退到上上个版本
versions[1].revision.revert()

# 刷新到数据库
obj.refresh_from_db()
assert obj.name == "version 1"

# 回退到上一个版本
versions[0].revision.revert()

# Check the model instance has been reverted.
obj.refresh_from_db()
assert obj.name == "version 2"
```

#### 删除恢复

```py
pk = obj.pk
obj.delete()

# Revert the second revision.
versions[0].revision.revert()

# Check the model has been restored to the database.
obj = YourModel.objects.get(pk=obj.pk)
assert obj.name == "version 2"
```

#### 源数据

```py
# Check the revision metadata for the first revision.
assert versions[1].revision.comment == "Created revision 1"
assert versions[1].revision.user == request.user
assert isinstance(versions[1].revision.date_created, datetime.datetime)

# Check the revision metadata for the second revision.
assert versions[0].revision.comment == "Created revision 2"
assert versions[0].revision.user == request.user
assert isinstance(versions[0].revision.date_created, datetime.datetime)
```


## 高级用法

#### 视图

```py
import json  # 示例使用的JSON序列化包
from django.views import View  # Django视图或RDF视图均可
from reversion.views import RevisionMixin  # 导入视图混合
from django.forms.models import model_to_dict as django_mtd  # Django转数据类型的包


class MyView(RevisionMixin, View):
    ''' 你的视图 '''
    revision_manage_manually = False # 是否手动管理版本
    revision_using = None # 默认数据库 default

    def revision_request_creates_revision(self, request):
        ''' 所有方法均保留操作记录 '''
        return True

    def post(self, request):
        # >>> Step 1
        obj = YourModel(name="obj v1") # 这里是你的Model, 必须是注册过的Model
        obj.save() # 这里创建了新的YourModel实例
        # >>> Step 2
        obj.name = "obj v2"
        obj.save() # 对象变更
        # 查询记录
        obj_dict = django_mtd(obj)
        return HttpResponse(json.dumps(obj_dict), content_type="application/json")

    get = post

```

参数解释:
- `revision_manage_manually = False`: 若为`True` YourModel.save() 将不会生成记录
- `revision_using = None`(*'default'*) : 指定使用那个数据库来写入记录(可用于读写分离)
- `revision_request_creates_revision(request)` 方法 : 默认情况下忽略 `GET` / `HEAD` / `OPTIONS` 三个请求的历史记录写入, 覆盖该方法来决定如何生成记录

> 若不覆盖 **revision_request_creates_revision** 方法, 在 **忽略的请求**方式下 的历史记录 **不会创建**, 但数据变更会 **正常生效**

#### 中间件

将每个请求都加入的历史记录中, 则可以直接启用`RevisionMiddleware`中间件配置, `reversion.middleware.RevisionMiddleware` 加入 `settings.py` 的 `MIDDLEWARE` 配置项中.

> 注意: 会自动启用数据库事务, 需要结合业务场景做性能方面的考虑

- `RevisionMiddleware.manage_manually = False` : 若为`True` YourModel.save() 将不会生成记录
- `RevisionMiddleware.using = None` : 指定使用那个数据库来写入记录(可用于读写分离)
- `RevisionMiddleware.request_creates_revision(request)` 方法 : 覆盖该方法来决定如何生成记录
- `RevisionMiddleware.atomic = True` : 是否使用事务包裹请求 `transaction.atomic()`

**RevisionMiddleware.request_creates_revision** 覆盖的样例: 

```py
from reversion.middleware import RevisionMiddleware

class BypassRevisionMiddleware(RevisionMiddleware):

    def request_creates_revision(self, request):
        # Bypass the revision according to some header
        silent = request.META.get("HTTP_X_NOREVISION", "false")
        return super().request_creates_revision(request) and \
            silent != "true"
```

## 差异对比

![img]({{ site.baseurl }}/img/in-post/development/python/django/django-reversion/p3.png "...")

[`django-reversion-compare`](https://pypi.org/project/django-reversion-compare/)

*django-reversion-compare*库是基于*django-reversio*库定制的, 可以对模型修改的历史进行差异对比

#### 安装

- 安装模块 `pip install django-reversion-compare`.
- 注册到 `INSTALLED_APPS` 
- Admin修改继承的Admin类.

**注册:** *注意前后顺序*

```py
INSTALLED_APPS = (
    'django...',
    ...
    'reversion', # 基础模块
    'reversion_compare', # django-reversion扩展的对比模块
    ...
)
```

#### 修改继承


```py
from django.contrib import admin
from reversion_compare.admin import CompareVersionAdmin

from my_app.models import YourModel

@admin.register(YourModel) # 你的表
class YourModelAdmin(CompareVersionAdmin): # 这里由 VersionAdmin 改为 CompareVersionAdmin 即可
    pass
```

由 `reversion.admin.VersionAdmin` 父类改为 `reversion_compare.admin.CompareVersionAdmin` 即可


其他高级功能请点击[这里](https://pypi.org/project/django-reversion-compare/)了解


## 注意事项

- `Queryset.update()` 方法不会生成历史记录
- 每当注册新的`Model`时, 运行一次`./manage.py createinitialrevisions`