---
layout: post
title: "Lock your procedure"
date: 2015-02-18 20:33:12 +8000
categories: [MySQL]
---
<!-- datetime: 2015-02-18 20:33:12 -->
<!-- more -->
在MySQL里，我们使用存储过程，以保证在执行出错的时候，数据可以即时的回滚到上一个状态。

存储过程并不是线程安全的。假如存储过程调用频率比较频繁时，如果同一个存储过程在同一刻被意外的打码多次，就有可能出现问题。

比如说下面的过程

```sql
declare va int;
select v1 into va from table1;
update table2 set v2 = v2 + va;
```

如果同时执行两次，则可能出现第一个过程在select后，update之前，第二个过程也select完成了。这时再update就会出现数据异常。

解决这个问题，可以在过程中使用具有幂等性的方法。如依赖MySQL的UniqueKey，或者在存储过程中使用`update table2 set v2 = value2`这样的句式。

但是这样的过程比不是在任何一个环境下都能写出来的。所以，在无法实现具有幂等性的句式下，需要保证存储过程同一时间只能运行一次。这是我们需要对存储过程加锁。

加锁的方法：

```sql
GET_LOCK("<lock_name>", "<expire_seconds>")
```

如`GET_LOCK('a_lock_name', 60)`表示以`a_lock_name`这个名字做锁，期限是60秒。超过60秒锁自动释放。成果获得锁返回true，否则返回false。

解锁的方法：

```sql
RELEASE_LOCK("<lock_name>")
```

如`DO RELEASE_LOCK("a_lock_name")`表示显示的释放名字为`a_lock_name`的锁。

例：

```sql
create procedure `procedure_name` ()
top:BEGIN
if not (GET_LOCK(`lock_name`, 60)) then
    leave top;
end if;

body:BEGIN

-- Your SQL here

COMMIT;
end body;

DO RELEASE_LOCK(`lock_name`);
end top;
```
