---
title: Django中的级联删除
description: 在django中的一对多关系里，默认存在着级联删除的属性。本文介绍了相关的属性设置。
date: 2023-02-13
updated: 2023-02-13
slug: cascading-deletes-in-django

tags:
 - Django
categories:
 - 技术
---
在django中的一对多关系里，默认存在着级联删除的属性。
### 处理方法
如果不想删除班级的同时员工表也被一起删除，需要在员工的班级外键上增加``` null=True, on_delete=models.SET_NULL```
#### 局部代码如下：
``` python .models.py {linenos=table,hl_lines=[14],linenostart=1}
class NCoVStaff(models.Model):
    TEACHER = 1
    PRINCIPAL = 2
    TYPE_CHOICE = (
        (TEACHER, 'Teacher'),
        (PRINCIPAL, 'Principal'),
    )
    name = models.CharField(max_length=100)
    cellphone = models.TextField(blank=True, null=True)
    staff_type = models.SmallIntegerField(default=TEACHER, choices=TYPE_CHOICE)
    wechat_user = models.OneToOneField('nCoV.NCoVWeChatUser', related_name='ncov_staff')
    created_time = models.DateTimeField(auto_now_add=True)
    center = models.ForeignKey('centers.Center', related_name='ncov_staff_center')
    klass = models.ForeignKey('nCoV.NCoVKlass', related_name='ncov_staff_klass', null=True, blank=True, on_delete=models.SET_NULL)

    def __unicode__(self):
        return '{}'.format(self.name)


class NCoVKlass(models.Model):
    name = models.CharField(max_length=100)
    no_of_student = models.IntegerField()
    center = models.ForeignKey('centers.Center', related_name='ncov_classes')

    def __unicode__(self):
        return '{}'.format(self.name)
```

### ```on_delete``` 可选属性
- CASCADE 
级联删除。 Django 默认选项，删除父项的同时会把子项一起删除。
删除子项时，子项的 Model.delete() 不会被调用， 但是pre_delete 和 post_delete 信号 signals 还是会捕获到。

- PROTECT
删除子项时抛出 ProtectedError，属于 django.db.IntegrityError。

- SET_NULL
把子项值设为 null； 子项同时要设置 null=True。

- SET_DEFAULT
把子项值设为 default 值； 子项同时要设置 default='...'

- SET()
指定一个方法来处理，如:
``` python {linenos=table,hl_lines=[11],linenostart=1}
from django.conf import settings
from django.contrib.auth import get_user_model
from django.db import models

def get_sentinel_user():
    return get_user_model().objects.get_or_create(username='deleted')[0]

class MyModel(models.Model):
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.SET(get_sentinel_user),
    )
```
- DO_NOTHING
 什么都不干 一般会导致IntegrityError。