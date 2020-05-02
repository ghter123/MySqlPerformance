## 为什么要有隔离：

多个事务之间同时对同一行/同一范围的数据的读写可能会造成 一个事务内 对数据的读取出现以下三种问题。

+ 脏读（读取到没有提交的数据,将来被回滚）
+ 不可重复读（事务内重复读取同一条数据不一样）
+ 幻行（事务内重复读取同一范围数据行数增多）

事务的不同级别对应了上述三个问题的出现的可能性。

## Mysql的事务隔离级别

### Read Uncommitted

一个事务可以读取另一个事务没有提交的数据。

### Read Committed

一个事务只能读取到另一个事务提交过的数据。

### Repeateable Read（Mysql的默认隔离级别）

一个事务对同一数据的读取是可重复的（多次读到都一样）。

### Serializable

事务之间顺序执行。

### 隔离级别对应的读取问题可能性

隔离级别 | 脏读可能性 | 不可重复读可能性 | 幻行可能性  
-|-|-|-
Read Uncommitted | Yes | Yes | Yes |
Read Committed   | No  | Yes | Yes |
Repeatable Read  | No  | No  | Yes |
Serealizable     | No  | No  | No  |

## InnoDB并发控制优化

Read Uncommitted 隔离级别几乎不用，Serealizable实现方式是加排它锁，这等于放弃了事务的并发。其它两个隔离级别比较常用分别作为不同数据库的默认选项，在InnoDB数据引擎里面也对这两种场景做了无锁实现优化（这个优化并不是完全的）。

> MVCC（多版本并发控制）：每个数据在每个事务执行完之后都会有一个快照，相当于一个事务对应一个快照（其实这里事务执行一次很像git里的Commit一次）。

在InnoDB中MVCC的简单实现可以表述为：
```` typescript
// MVCC中一个版本号对应一个事务，事务的版本号随提交的时间递增。并且会对每条数据增加两个字段。
interface MVCCProperty {
    updatedVersion: number;
    removedVersion?: number;
}

// 假设这是一张用户表
class user {
    constructor(mVCCProperty:MVCCProperty,name:string,age:number) {
        this.mVCCProperty = mVCCProperty;
        this.name = name;
        this.age = age;
    } 
}

/** 事务中对user的增删改查可以表现为
Select
    1. transation.version >= user.updatedVersion
    2. transation.version < user.removedVersion || user.removedVersion == null
Insert
    1. user.updatedVersion = transation.version
Delete
    1. user.removedVersion = transation.version
Update 
    1. user.removedVersion = transation.version
    2. user = new User({
            updatedVersion: transation.version,
            removedVersion: null
        },user.name,user.age)
**/
````