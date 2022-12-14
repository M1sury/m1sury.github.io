## 什么是索引

网上大多数的例子都会以书为例子，就像看书一样，你想快速查询到你想看的部分内容，一定不会选择随机的翻页或者一页一页找。一定是会看目录寻找你想看的内容。换到数据库中，索引就像是一本书一样，**索引就是MySQL 数据的目录**

## 索引分类

### 按数据结构分类

* B+Tree索引(**主流**)
* Hash索引
* Full-Text索引

### 按物理存储分类

* 聚簇索引：叶子节点存放完整数据
* 二级索引：叶子节点存放主键值

### 按字段特性分类

* 主键索引：建立在主键字段的索引，一个表最多一个主键索引，**不允许有空值**
* 唯一索引：建立在 UNIQUE 字段上的索引，一个表可以有多个索引，索引列的值必须唯一，**但是允许有空值**
* 普通索引
* 前缀索引：对字符类型字段的前几个字符建立的索引，而不是整个字段，可以建立在`char,varchar,binary,varbinary`类型的列上

在创建表时，InnoDB 存储引擎会根据不同的场景选择不同的列作为索引：

* 有主键，默认使用主键作为聚簇索引的索引键
* 没索引，选择第一个不包含 NULL 值的唯一列作为聚簇索引的索引键
* 以上两种都没有，InnoDB 将自动生成的一个隐式自增id 作为聚簇索引的索引键

其他索引都属于二级索引或非聚簇索引。**创建主键索引和二级索引默认使用的 B+Tree 索引**

创建一个用户表，id为主键

```sql
CREATE TABLE `user`  (
  `id` bigint NOT NULL,
  `username` varchar(255)  DEFAULT NULL,
  `password` varchar(255) DEFAULT NULL,
  `enable` bigint NOT NULL,
  PRIMARY KEY (`id`) USING BTREE
) CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

INSERT INTO `interview`.`user`(`id`, `username`, `password`, `enable`) VALUES (1, 'test', 'test', 1);
INSERT INTO `interview`.`user`(`id`, `username`, `password`, `enable`) VALUES (2, '小明', 'password', 0);
INSERT INTO `interview`.`user`(`id`, `username`, `password`, `enable`) VALUES (3, '小红', 'xiaohong', 1);
```

这些数据，存储在 B+Tree 索引时是什么样子的？

B+Tree 是一种多叉树，叶子节点才存放数据，非叶子节点只存储索引，并且每个节点的数据都是**按主键顺序存放**，每一层父节点的索引值都会出现在下层子节点的索引值中，因此叶子节点会存储所有的索引值信息。每一个叶子节点都会指向下一个叶子节点，形成一个链表

![未命名文件 (1)](https://my-lottery.oss-cn-shanghai.aliyuncs.com/lottery/未命名文件 (1).png)

**通过主键索引查找用户数据过程**

```sql
select * from user where id = 5
```

B+ Tree 会自顶向下逐层进行查找：

* 将5 和根节点的索引数据(1,10,20)比较，5在 1 和 10 之间，找到第二层索引数据(1,4,7)
* 在第二层索引数据中查询，因为5在 4和7之间，找到第三层索引数据(4,5,6)
* 在叶子节点的索引数据(4,5,6)进行查找，找到了索引值为5的行数据

数据库的索引和数据都是存在硬盘的，可以把读取一次节点当作一次磁盘 IO操作，上面整个查询过程经历了三个节点，继续了3次磁盘IO

B+Tree存储千万级的数据只需要`3~4`层高度就可以满足，意味着从千万级的表查询目标数据最多需要`3~4`次IO操作。**所以相比于B 树 和二叉树来说，最大的优势在于查询效率很高，即使在数据量很大的情况，查询一个数据的磁盘IO依然很少**



**如果二级索引查询用户数据的过程**

```sql
# 创建索引
# 假设enable代表点赞数,如果用作开关的话，最好不要创建索引，否则会用不到索引，辨识率太低
create index idx_enable on user(`enable`)
```

![image-20221013103337627](https://my-lottery.oss-cn-shanghai.aliyuncs.com/lottery/image-20221013103337627.png)

![未命名文件 (2)](https://my-lottery.oss-cn-shanghai.aliyuncs.com/lottery/未命名文件 (2).png)

主键索引的 B+Tree 和二级索引的 B+Tree 区别如下

* 主键索引的 B+Tree 叶子节点存储的是实际数据，所欲的用户记录都存在主键索引的 B+Tree 叶子节点中
* 二级索引的 B+Tree 的叶子节点存放的是主键值，而不是实际数据

```sql
select * from user where enable = 2
```

上面这条SQL 语句会检索二级索引的 B+Tree的索引值，找到对应的叶子节点，然后获取主键值，然后再通过主键索引的 B+Tree 查询到对应叶子节点，获取全部数据。这个过程叫**回表**，就是说需要查两个 B+ Tree 才能查到数据

```sql
select enable from user where enable = 2
```

如上条语句，查询的数据能在二级索引的叶子节点中查询到，这时就不需要再查主键索引值了，**这种在二级索引的 B+ Tree就能查询到结果的过程叫 覆盖索引**

### 按字段个数分类

* 单列索引：建立在单列上的索引
* 联合索引：建立在多列上的索引

```sql
# 只是演示，开发阶段不可能用password和enable做索引
CREATE INDEX index_pwd_enable ON user(password, `enable`);
```

联合索引的非叶子节点是用索引的多个字段作为 B+ Tree 的key值，当联合索引查询数据，遵循**最左前缀法则**，先按`password`字段比较，在`password`相同的情况下，再按`enable`字段比较

如果创建了一个`(a,b,c)`联合索引，如果查询条件是这几种，可以匹配上联合索引:

* where a = 1;
* where  a =1 and b=2 and c=3;
* where a=1 and b=2;

需要注意，因为有**查询优化器**的存在，所以 a 字段在where子句的顺序并不重要

![image-20221013105556851](https://my-lottery.oss-cn-shanghai.aliyuncs.com/lottery/image-20221013105556851.png)

但是，如果查询条件是以下几种，不符合最左前缀法则，就无法匹配上联合索引，索引就会失效

* where b=2;
* where c=3;
* where b=2 and c=3;

先按`a`排序，在 a 相同的情况下再按 b 排序，b 和 c 是局部相对有序的。

并且，**联合索引的最左前缀法则，在遇到范围查询(大小于，between，like等)就会停止匹配，使用范围查询的列可以用到索引，范围列后面的列没办法用到索引**

`select * from test where a>1 and b=2`只能用到 a字段的索引，B+Tree找到第一个满足条件的主键值后，还需要判断其他条件是否满足，是在联合索引里面判断，还是在主键索引判断？

* 在 MySQL 5.6之前，只能从主键值里面一个个回表，到**主键索引**上找出索引行，再比对其他条件是否满足
* 5.6版本引入的**索引下推优化**，可以在**联合索引遍历过程中，对联合索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数**

当查询语句的执行计划总，出现了 Extra 为`Using index condition` 就说明使用了索引下推优化

**建立联合索引时，要把区分度大的字段排在前面，区分度大的字段越有可能被更多SQL使用到**

## 什么时候需要创建索引

索引虽然可以提高查询速度，但是也不是万金油，也是有缺点的：

* 索引也需要占用物理空间，数据量大，占用空间也大
* 创建索引和维护索引需要耗费时间
* 会降低表的增删改效率，每次增删改索引，B+树为了维护索引有序，需要动态维护

### 什么时候适合建索引

* 字段有唯一性限制，类似于用户ID
* 经常用于 where查询条件的字段
* 经常`group by` 和`order by`的字段，这样查询的时候就不需要再去做排序，B+Tree中的记录都是排序好的

### 什么时候不适合建索引

* `Where`,`Group By`,`Order By`用不到的字段，索引的价值是快速定位，既然用不到也不需要建，并且索引是会占用物理空间的
* 字段辨识度不高，就像上面那个例子，用`enable`作为索引的字段是不明智的，一个开关可能的值也就只有 1 和 0，如果表中记录平均分布，搜索哪个值都能得到大概一半数据，**就算建立索引，查询优化器，会发现某个值在列中的百分比很高，会忽略索引，进行全表扫描**
* 经常更新的字段不用创建索引，**由于索引字段频繁修改，需要维护B+ Tree 的有序性，就要频繁重建索引，这个过程很影响数据库性能**

## 防止索引失效

用上了索引并不意味着查询的时候就会使用索引，需要避免写出索引失效的查询语句，否则查询效率很低

发生失效的情况：

* 模糊匹配：`like %xx%`和`like %xx`，都会造成索引失效
* 函数：如果在查询条件中对索引做了计算、函数、类型转换操作，都会索引失效
* 联合索引，不遵守最左前缀法则也会索引失效
* Where子句中，如果OR前的条件列是索引列，OR后的条件列不是索引列，也会失效

执行计划中的 type 是描述查找数据使用的扫描方法，常见的扫描类型：

* ALL（全表扫描）
* index（全索引扫描）
* range（索引范围扫描）
* ref（非唯一索引扫描）
* eq_ref（唯一索引扫描）
* const（结果只有一条的主键或唯一索引扫描）

**最好在开发期间，把写的SQL看一下执行计划，避免索引失效**

## MySQL为什么采用 B+Tree作为索引

> 这个问题最直接的回答当然是因为**快**了

因为 MySQL 的数据是存储在磁盘中的，当通过索引查找某行数据的适合，需要先从磁盘读取索引到内存，再通过索引从磁盘中找到某行数据，然后读取到内存中。期间会发生多次磁盘IO，IO次数越多，消耗时间也就越久。**并且，MySQL 是支持范围查询的，所以索引的数据结构要支持高效的查询某一个记录，也要高效的执行范围查询**

我们可以先分析一下常用的数据结构：

### 数组

数组这种数据结构，**数组是内存中一块连续的内存空间**，并且有下标，如果知道这个元素的下标，可以在`O(1)`的时间复杂度获取到元素

对于一个有序数组，查找可以通过二分查找，时间复杂度为 `O(logn)`，对删除和修改的效率也很高，**但是，**在新增的时候，尤其是在数组中间插入数据的时候，需要把插入的目标位置后面的所有元素都往后挪动位置，才能插入新的数据，涉及了数组的复制操作，**不仅需要额外开辟内存，复制数据消耗的时间也很长**

**所以，从插入数据的层面上看，数组不适合作为MySQL索引的数据结构**

### 哈希表

哈希表是一个 `key-value`的数据结构，底层采用` 数组+链表`的形式实现，是将 key 通过哈希函数计算出一个数字，然后用这个数字作为数组的下标，把value存放在对应下标的数组中。

链表的作用主要是为了解决**哈希冲突**的，对于不同的 key，有可能经过哈希函数，计算出相同的值，比如`通话`和`重地`在 java 中的 hashcode就是一样的，意味着同一个数组下标要存放两个元素，这时就需要通过链表把两个元素串联起来

**虽然哈希表对于删除、查找、更新、插入操作**，时间复杂度都是`O(1)`，但是在使用 MySQL 中，经常遇到查找某个范围内的值，这时候，哈希表就比较耗时了

### 二叉树

> 二叉树的特点是一个节点的左子树的所有节点都小于根节点，右子树的所有节点都大于根节点

这样就可以使用二分查找法，并且二叉树也不需要内存连续，插入新节点，可以放在任何位置

![image-20221013114257606](https://my-lottery.oss-cn-shanghai.aliyuncs.com/lottery/image-20221013114257606.png)

这个**看起来**也是一个可以作为索引的数据结构，但是如果在一个极端情况下，**每次插入的元素都是二叉树的最大元素，二叉树就会退化成一条链表，查找数据的时间复杂度就会变成O(n)**

![image-20221013114537737](https://my-lottery.oss-cn-shanghai.aliyuncs.com/lottery/image-20221013114537737.png)

由于树是存储在磁盘中的，访问每个节点，都对应一次磁盘IO，也就是**树的高度等于每次查询数据时磁盘IO操作的次数**，并且不能进行范围查询，所以不适合作为数据库的索引结构

### 自平衡二叉树(AVL树)

为了解决二叉树在极端情况下退化成链表的问题，后面有人提出了平衡二叉树(AVL树)

主要是在二叉查找树的基础上增加了一些条件约束，**每个节点的左子树和右子树的高度差不能超过1**，节点的左子树和右子树仍然为平衡二叉树，这样查询操作的时间复杂度就会一直维持在`O(logn)`

除了 AVL 树，还有很多自平衡的二叉树，比如红黑树，也是通过一些约束条件来达到自平衡，不过红黑树的约束条件比较复杂

**不管是 AVL 树还是红黑树，都会随着插入的元素增多，而导致树的高度变高，意味着磁盘IO操作次数多，影响整体查询效率**

因为它们都是二叉树，也就是每个节点只能保存两个子节点，如果把二叉树变成 M叉树呢？

**当树的节点越多的时候，并且树的分叉书越大，M 叉树的高度会远小于二叉树的高度**

### B树

 为了解决降低树的高度问题，就出现了 B 树，不再限制一个节点只能有两个节点，允许有 M(M>2)个子节点，降低了树的高度

B 树的每一个节点最多可以包括 M 个子节点，M称为 B 树的阶，所以 B 树就是一个多叉树。

假如 M =3，那么就是一棵 3阶的 B 树，特点就是每个节点最多有2个(M -1个)数据和最多有 3 个(M个)子节点，超过这些要求，就会分裂节点，

![image-20221013142816479](https://my-lottery.oss-cn-shanghai.aliyuncs.com/lottery/image-20221013142816479.png)

![image-20221013142850441](https://my-lottery.oss-cn-shanghai.aliyuncs.com/lottery/image-20221013142850441.png)

![image-20221013142906496](https://my-lottery.oss-cn-shanghai.aliyuncs.com/lottery/image-20221013142906496.png)

一颗 3 阶的 B 树在查询叶子节点中的数据时，由于树的高度时 3，所有在查询过程中会发生 3次磁盘IO

如果同样的节点数量在二叉树的场景下，树的高度就会很高，所以B树在数据查询中比平衡二叉树效率要高

但是 B 树的每个节点都包含数据（索引+记录），而用户的记录数据的大小很有可能远远超过索引数据，这就需要花费更多的磁盘IO次数来读到**有用的索引数据**

我们在查询位于底层的某个节点过程中，不是**目标记录节点的记录数据**会从磁盘加载到内存中，但是这些记录数据时没用的，我们只想读取这些节点的索引数据来做比较查询，非目标记录节点的记录数据对我们是没用的，不仅增多磁盘IO次数，也占用内存

并且使用 B 树来做范围查询，需要使用中序遍历，会设计多个节点的磁盘IO，导致整体速度下降

### B+树

B + 树就是对 B 树做了一个升级，MySQL中索引的数据结构就是采用了 B+ 树

![image-20221013143533591](https://my-lottery.oss-cn-shanghai.aliyuncs.com/lottery/image-20221013143533591.png)

B+树和 B 树的区别：

* 叶子节点(最底部的节点)才会存放实际数据，非叶子节点只会存放索引
* 所有的索引都会在叶子节点出现，叶子节点之间构成一个有序链表
* 非叶子节点的索引也会同时存在子节点中，并且是在子节点中所有索引的最大(或最小)
* 非叶子节点中有多少个子节点，就有多少个索引

**B+ 树的非叶子节点不存放实际的记录数据，仅存放索引，因此数据量相同的情况下，和存储索引和记录的B 树，B+树的非叶子节点可以存放更多的索引，查询底层节点的磁盘IO次数更少**

B+树所有叶子节点间有一个链表进行连接，这种设计对范围查找非常有帮助

### InnoDB中的 B+树

* InnoDB 中的 B+树，叶子节点之间是用**双向链表**连接，好处是**既能向右遍历，也能向左遍历**

* B+ 树节点内容是数据页，数据页里存放了用户的记录以及各种信息，**每个数据页默认大小是16KB**