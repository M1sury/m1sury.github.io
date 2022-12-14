## MySQL的锁

### 按照锁的粒度划分

* 全局锁：锁定数据库中的所有表

```sql
# 使用全局锁
flush tables with read lock
# 整个数据库就处于只读状态，其他操作比如insert,delete,update,alter等都会被阻塞


# 释放全局锁
unlock tables
```

**或者会话断开了，全局锁也会被释放**

**全局锁主要应用于全库逻辑备份，这样在备份数据库期间，不会因为数据或表结构更新，出现备份文件的数据与预期的不一样**

* 表级锁：每次操作锁定整张表

* 行级锁：每次操作锁住对应的行数据

### 表级锁

MySQL中 表级别的锁有几种：

* 表锁
* 元数据锁（MDL）

* 意向锁
* AUTO-INC 锁

#### 表锁

表锁除了**会限制别的线程的读写外，也会限制本线程接下来的读写操作**。

```sql
# 表级别的共享锁，读锁
lock tables t_student read;

# 表级别的排他锁，写锁
lock tables t_student write;
```

如果本线程对该表加了**共享表锁**，本线程如果要对该表执行写操作，也会被阻塞，其他线程进行写操作也会阻塞，直到锁被释放

```sql
unlock tables
```

#### 元数据锁

不需要显示的使用 MDL，因为当对数据库表进行操作时，会自动给这个表加上 MDL

* 对一张表进行 CRUD 操作时，加的时 MDL读锁
* 对一张表做结构变更操作的时候，加的是 MDL写锁

MDL 是为了保证当用户对表执行 CRUD 操作时，防止其他线程对这个表结构做了变更

#### 意向锁

* 在使用 InnoDB 引擎的表里对某个记录加上**共享锁**之前，需要先在表级别加上一个**意向共享锁**

* 在使用 InnoDB 引擎的表里对某些记录加上独占锁之前，需要先在表级别加上一个**意向独占锁**

**意向锁的目的时为了快速判断表里是否有记录被加锁**

#### AUTO-INC锁

在为某个字段声明`Auto_increment`属性时，之后可以在插入数据时，可以不指定该字段的值，数据库自动给该字段赋值递增的值，主要是通过`Auto_inc`锁实现的

**一个事务在持有 AUTO-INC 锁的过程中，其他事务如果要向该表插入语句都会被阻塞，从而保证插入数据时，被`AUTO_INCRMENT`修饰的字段的值是连续递增的**

> 不是重点，先略过

### 行级锁

> InnoDB 引擎是支持行锁的，MyISAM引擎不支持行级锁

普通的 select 语句是不会对记录加锁的，因为属于快照读。如果要在查询时对记录加行锁，可以使用下面两种方式

，这种查询会加锁的语句称为**锁定读**

```sql
# 对读取的记录加共享锁
select … lock in share mode;

# 对读取的记录加独占锁
select … for update;
```

行锁的类型有三种：

* Record Lock：记录锁，也就是仅仅把一条记录锁上
* Gap Lock：间隙锁，锁定一个范围，但是不包括记录本身
* Next-Key Lock：临键锁，Record Lock+Gap Lock的组合。锁定一个范围，**并且锁定记录本身**

#### Record Lock

> Recrod Lock 称为记录锁，锁住的是一条记录，而且记录锁是有 S 锁和 X 锁之分的：

* 当一个事务对一条记录加 S 型记录锁后，其他事务也可以继续对该记录加 S 型记录锁，但是不可以对该记录加 X 型记录锁
* 当一个事务对一条记录加了 X 型记录锁后，其他事务既不可以对该记录加 S 型记录锁，也不可以对该记录加 X 型记录锁

**读读共享，读写互斥。写读互斥，写写互斥**

**当事务提交后，事务过程中生成的锁都会被释放**

#### Gap Lock

> Gap Lock称为间隙锁，只存在于可重复读隔离级别，目的是为了解决可重复读隔离级别下幻读的现象

假设，表中有一个范围 id 为(3,5)间隙锁，那么其他事务就无法插入 id = 4 这条记录了，这样就有效防止幻读现象的发生

**间隙锁之间是兼容的，即两个事务可以同时持有包含共同间隙范围的间隙锁，并不存在互斥关系，因为间隙锁的目的是防止插入幻影记录而提出的**

#### Next-Key Lock

> 称为临键锁，是 记录锁+间隙锁的组合，锁定一个范围，并且锁定记录本身

假设表中有一个范围id 为(3,5)的临键锁，其他事务既不能插入 id = 4 记录，也不能修改 id = 5 的记录

**所以临键锁既能保护该记录，又能阻止其他事务将新纪录插入到被保护记录前面的间隙中**

如果一个事务获取了 X 型的临键锁，那么另外一个事务在获取相同范围的 X 型的 临键锁，是会被阻塞的

**MySQL记录锁+间隙锁可以防止删除操作导致的幻读**