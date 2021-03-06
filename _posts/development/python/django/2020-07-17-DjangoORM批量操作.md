---
title: "Django ORM 常用操作"
subtitle: "Django | ORM | model批量操作 | 连接管理 "
layout: post
author: "luoruiqing"
header-style: text
tags:
  - Django

---


## 批量操作 


#### 插入 `Model.bulk_create`

```python
# 多个数据实例
insert_data = [UserModel(age=index) for index in range(10)]
# 调用bulk_create方法批量写入
UserModel.objects.bulk_create(insert_data)
```

#### 更新 `QuerySet.update`

```python
# 将所有名称包含"王"的所有名称改为李四
UserModel.objects.filter(name__contains='王').update(name='李四')
```

#### 删除 `QuerySet.delete`


```python
# 删除名称包含"王"字在内的所有用户
UserModel.objects.filter(name__contains='王').delete()
```


## 连接

#### 连接保持

```
#...
```


#### 手动关闭



#### 连接池


```
# ...
```