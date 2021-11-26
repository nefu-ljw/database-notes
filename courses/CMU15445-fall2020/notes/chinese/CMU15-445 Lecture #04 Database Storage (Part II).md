## CMU15-445 Lecture #04: Database Storage (Part II)

课程链接：[15-445/645 Database Systems (Fall 2020)](https://15445.courses.cs.cmu.edu/fall2020)

本文由 [nefu-ljw](https://github.com/nefu-ljw) 翻译于Notes：[https://15445.courses.cs.cmu.edu/fall2020/notes/04-storage2.pdf](https://15445.courses.cs.cmu.edu/fall2020/notes/04-storage2.pdf)

所有Notes已同步更新于我的github仓库：[https://github.com/nefu-ljw/database-notes](https://github.com/nefu-ljw/database-notes)

**目录**

* [1\. Data Representation](#1-data-representation)
* [2\. Workloads](#2-workloads)
* [3\. Storage Models](#3-storage-models)

### 1. Data Representation

元组中的数据本质上只是字节数组。DBMS需要知道如何解释这些字节来派生属性值。数据表示模式是DBMS存储值的字节。

有五种高级数据类型可以存储在元组中：整数、可变精度数字、定点精度数字、可变长度数据和日期/时间。

**Integers** (整数)

大多数DBMS使用IEEE-754标准指定的“本机”(native)C/C++类型存储整数。这些值是固定长度。

例子: INTEGER, BIGINT, SMALLINT, TINYINT

**Variable Precision Numbers** (可变精度数字)

这些是不精确、变精度的数值类型，它们使用IEEE-754标准指定的“本机”C/C++类型。这些值也是固定长度。

对变精度数字的运算比对任意精度数字的运算速度更快，因为CPU可以直接对它们执行指令。但是，由于某些数字不能精确表示，因此在执行计算时可能会出现舍入误差。

例子: FLOAT, REAL

**Fixed-Point Precision Numbers** (定点精度数字)

这些是具有任意精度和小数位数的数值数据类型。它们通常以精确的、可变长度的二进制表示形式存储(几乎就像一个字符串)，并带有附加的元数据，这些元数据将告诉系统数据的长度和小数应该在哪里等信息。

当舍入误差不可接受时，将使用这些数据类型，但DBMS会为获得此精度而付出性能代价。

例子: NUMERIC, DECIMAL

**Variable-Length Data** (可变长度数据)

它们表示任意长度的数据类型。它们通常与跟踪字符串长度的header一起存储，以便于跳转到下一个值。它还可以包含数据的校验和。

大多数DBMS不允许元组超过单个页面的大小。那些确实将数据存储在特殊的“溢出”(overflow)页面上并使元组包含对该页面的引用。这些溢出页可以包含指向其他溢出页的指针，直到可以存储所有数据为止。

有些系统允许您将这些大值存储在外部文件中，然后元组将包含指向该文件的指针。例如，如果数据库存储照片信息，则DBMS可以将照片存储在外部文件中，而不是让它们占用DBMS中的大量空间。这样做的一个缺点是DBMS不能操作该文件的内容。因此，不存在持久性或事务保护。

例子: VARCHAR, VARBINARY, TEXT, BLOB

**Dates and Times** (日期/时间)

日期/时间的表示形式因系统不同而不同。通常，这些时间表示为自Unix时代以来的某个单位时间(微/毫秒)。

例子: TIME, DATE, TIMESTAMP

**System Catalogs** (系统目录)

为了使DBMS能够破译元组的内容，它维护一个内部目录来告诉它有关数据库的元数据。元数据将包含有关数据库有哪些表和列的信息，以及它们的类型和值的排序。

大多数DBMS以用于其表的格式将其目录存储在自身内部。他们使用特殊代码来“引导”(bootstrap)这些目录表。

### 2. Workloads

数据库系统有许多不同的工作负载。关于工作负载，我们指的是系统必须处理的请求的一般性质。本课程将重点介绍两种类型：联机事务处理(OLTP)和联机分析处理(OLAP)。

**OLTP: Online Transaction Processing**

OLTP工作负载的特点是快速、短时间运行操作、单次操作单个实体的简单查询和重复操作。OLTP工作负载通常处理的写操作多于读操作。

亚马逊店面就是OLTP工作负载的一个示例。用户可以向购物车中添加物品，也可以进行购买，但这些操作只会影响他们的账户。

**OLAP: Online Analytical Processing**

OLAP工作负载的特点是长时间运行、复杂的查询、对大部分数据库的读取。在OLAP工作区中，数据库系统从OLTP端收集的现有数据中分析和派生新数据。

OLAP工作负载的一个示例是亚马逊计算这些地理位置在一个月内购买最多的五种商品。

**HTAP: Hybrid Transaction + Analytical Processing**

最近流行的一种新型工作负载是HTAP，它就像是试图在同一数据库上同时执行OLTP和OLAP的组合。

![image-20211126152422017](https://i.loli.net/2021/11/26/rEZTAFSNQWtxHdM.png)

### 3. Storage Models

在页面中存储元组有不同的方法。到目前为止，我们假设了n元存储模型(n-ary storage model)。

**N-Ary Storage Model (NSM)**

在n元存储模型中，DBMS在单个页面中连续存储单个元组的所有属性，因此NSM也称为“行存储”(row store)。此方法非常适合请求插入频繁且事务往往只操作单个实体的OLTP工作负载。它非常理想，因为只需一次提取即可获得单个元组的所有属性。

优点：

- 快速插入、更新和删除。
- 适用于需要整个元组的查询。

缺点：

- 不适用于扫描表的大部分和/或属性子集。这是因为它会获取处理查询不需要的数据，从而污染缓冲池。

![image-20211126153010847](https://i.loli.net/2021/11/26/cNxFXAVSqte4MrQ.png)

**Decomposition Storage Model (DSM)**

在分解存储模型中，DBMS将所有元组的单个属性(列)连续存储在一个数据块中。因此，它也被称为“列存储”(column store)。此模型非常适合具有许多只读查询的OLAP工作负载，这些只读查询对表属性的子集执行大型扫描。

优点：

- 减少查询执行期间浪费的工作量，因为DBMS只读取该查询所需的数据。
- 启用更好的压缩，因为同一属性的所有值都是连续存储的。

缺点：

- 由于元组拆分/缝合，点查询、插入、更新和删除的速度较慢。

要在使用列存储时将元组重新组合在一起，有两种常见的方法：最常用的方法是定长偏移(fixed-length offsets)。假设属性都是固定长度的，DBMS可以计算每个元组的属性偏移量。然后，当系统需要特定元组的属性时，它知道如何从offest跳到文件中的那个位置。要容纳可变长度的字段，系统可以填充字段，使其长度相同，也可以使用字典，该字典采用固定大小的整数并将该整数映射到该值。

一种不太常见的方法是使用嵌入式元组ID(embedded tuple ids)。在这里，对于列中的每个属性，DBMS都存储一个元组ID(例如：主键)。然后，系统还会存储一个映射，告诉它如何跳转到具有该ID的每个属性。请注意，此方法具有很大的存储开销，因为它需要为每个属性条目存储一个元组ID。

![image-20211126153425453](https://i.loli.net/2021/11/26/mYgQwFq79yCIdOf.png)
