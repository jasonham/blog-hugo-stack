---
title: 如何在Django中改变collation
description: 在Django（mysql）中把表格的Collate改成utf8mb4_bin
date: 2023-03-18 20:33:31
slug: how-to-change-collation-with-django

tags:
 - Django
 - MySQL
categories:
 - 技术
---
## 问题简述
默认情况下，对于UTF-8数据库，MySQL将使用 utf8_general_ci来做为数据库的collation。这导致所有字符串不会区分大小写。即，"Abcd"与"abcD"在数据库级别被视为相等。如果对字段有唯一约束，则尝试将"aa"和 "AA"插入同一列将是非法的。如果要对特定的列或表进行区分大小写的比较，需要更改该列或表以使用 utf8_bin的collation。Django没有提供在模型定义上进行设置的方法。

而collation的设置可以分别在数据库级别，表级别，列级别进行设置，这里主要说说后两种
### 表级别
- 用Django的south组件中的schemamigration生成migrations文件
- 修改migrations文件，在表创建完成的语句后增加修改表的collation语句
``` python 0009_auto__add_ncovklassconfirm.py {linenos=table,hl_lines=[7, 8],linenostart=1}
class Migration(SchemaMigration):

    def forwards(self, orm):
        # Adding model 'NCoVKlassConfirm'
        ...
        # Change collation
        db.execute(
            "ALTER TABLE nCoV_ncovklassconfirm CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;")
        ...
    def backwards(self, orm):
```
### 列级别
- 用Django的south组件中的schemamigration生成migrations文件
- 修改migrations文件，在表创建完成的语句后增加修改列的collation语句
``` python 0003_auto__add_field_ncovwechatuser_parent.py {linenos=table,hl_lines=["13-14"],linenostart=1}
class Migration(SchemaMigration):

    def forwards(self, orm):
        # Adding field 'NCoVWeChatUser.parent'
        # db.add_column(u'nCoV_ncovwechatuser', 'parent',
        #               self.gf('django.db.models.fields.related.ForeignKey')(related_name='children', null=True, to=orm['nCoV.NCoVWeChatUser']),
        #               keep_default=False)
        # Manual adding 'NCoVWeChatUser.parent'
        db.add_column(u'nCoV_ncovwechatuser', 'parent_id',
                    self.gf('django.db.models.fields.CharField')(max_length=128, null=True),
                    keep_default=False)
        # Alter collation
        db.execute(
            "ALTER TABLE nCoV_ncovwechatuser MODIFY parent_id VARCHAR(128) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;")
        # Add Constraint
        db.execute(db.foreign_key_sql('nCoV_ncovwechatuser', 'parent_id', 'nCoV_ncovwechatuser', 'openid'))

    def backwards(self, orm): 
```