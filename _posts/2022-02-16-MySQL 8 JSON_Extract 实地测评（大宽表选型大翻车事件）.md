---
layout: post
title: "MySQL 8 JSON_Extract 实地测评（大宽表选型大翻车事件）"
date: 2022-02-16 09:42:58 +8000
categories: [MySQL]
---

记录投管家做数据存储的选型时遇到的问题和各个业务场景下的调研结果。

##  表结构设计

我们先用一个简单的例子来假设，假设我们有一个ad_report表，其中period和ad_id是表的数据维度，display，click，cost是数据指标，这样我们就有了一个可以存储每个ad_id每天的数据的Report表，表大概长这样：  
![image](https://user-images.githubusercontent.com/3870517/154229777-2a5b76a9-b907-448e-8e72-9021d9170b39.png)

Report表中指标数据的存储有几种方式：
1. 基本格式存储：每个指标存1列，有多少指标MySQL就有多少列
2. KV列转行存储：用key-value的方式做列转行存储
3. JSON存储：用一个JSON类型的字段存储

### 基本格式存储

优点：
* 最符合范式设计了，每个字段都有对应的定义、约束、注释，SQL写起来也最简单，大部分时候应该是最常见的选型。

缺点：
* 如果列非常多，性能就会大大下降，还有可能会出现buffer爆炸的情况。官方给到的MySQL表列最大数是4096列，实际测试发现大几百列就会出现各种奇怪的问题。

### KV行转列存储

相当于把原本的display，click，cost这3列转成行来存储，原本一行可以存储的数据现在用3行拉存储，如下图，我们用field=display/click/cost来分别指代这3个指标，value则存储对应的值。  
![image](https://user-images.githubusercontent.com/3870517/154230641-d91f2bf8-0c34-447f-bcb5-2d91c53640cb.png)

优点：
* 如果列非常多，但是大部分情况下大部分列都是没有数据的，并且业务每次查询只查少量的几个指标，那种这种方法存储会得到比较好的压缩效果。

缺点：
* 行数很容易增长爆炸，如果本身report数据量就很大，指标又多，列转行存储会导致RowCount爆炸
* value的数据类型会受到限制，如果有的指标是浮点数，有的是整数，有的是Decimail，这里就只能用varchar来存储，缺少约束
* 如果要一次查询大量指标，SQL写起来很困难，并且要做行转列查询，SQL复杂，且效率不高（猜的）

### JSON存储

所有的指标都存在一个字段里，比如叫payload

优点：
* 如果report的指标经常要更改（比如新增指标），都放JSON里就不需要频繁调整表结构
* 如果指标很多，但是大部分时候是空，用JSON可以很好的压缩数据
* 如果某个指标需要做索引，可以直接用JSON_VALUE来添加索引或者用generated always as来增加索引用的字段

缺点：
* JSON的结构需要自己控制，如果约束不到位可能会导致出现奇怪的数据在表里
* 如果字段数少，并且大部分字段都是有值的，那用JSON存反而更占空间得不偿失
* 查询的时候使用JSON_Extract取出JSON里的值，会产生大量的Decode操作，CPU开销比较高性能比较差（猜的）
* JSON_Extract如果没有值会直接给NULL，如果要进行指标运算还要额外增加IFNULL操作，SQL书写比较复杂

![image](https://user-images.githubusercontent.com/3870517/154231827-e2512929-7aea-4277-93c5-093bb7e561dd.png)

## 选型

目前线上业务选择的是方法一：基本格式存储，原因是：
1. BI Report每次都要查询大量字段，行列转换不太可行
2. JSON Decode性能堪忧，如果客户需要检索3个月的数据，就会从数据库中翻出几百万行的数据，这些数据都要进行JSON Decode才能取出里面的值进行运算，大量的JSON操作会导致MySQL性能瓶颈（猜的）

## 遇到的问题

### 字段太多查询不了

业务实际的表里有大概300+个指标字段，在经过JOIN很多表，GROUP BY，ORDER BY后，报错了`ERROR 1118 (42000): Row size too large (> 8126).`
研究发现这是在做GROUP BY或者ORDER BY的操作时，需要建立Temp Table，因为字段太多，超过了Temp Table的上限所以报错。要解决这个问题需要修改ini文件。  
<img width="759" alt="EEFD5B4C-AEBF-43E5-9A72-D7FCB9ED5381" src="https://user-images.githubusercontent.com/3870517/154238251-1ee65e35-6e21-4054-90e9-600023cb3770.png">
修改ini文件的暂时就不考虑，GROUP BY和ORDER BY是无法避免的，所以实际上不管用上面哪种方法存储都无法解决这个问题。最后从业务侧要求减少每次查询获取的指标数来解决，比如说每次查询选取的指标数不超过30个。

### 稀疏矩阵浪费资源

整个表300多个字段，95%以上的行实际有值的指标不过七八个，大量指标都是0，存储在MySQL中产生相当大的浪费，表中大概167w条数据的情况下，表size就达到了5g之多  
<img width="246" alt="image" src="https://user-images.githubusercontent.com/3870517/154243528-276cacb5-12ca-4dd8-92bd-3dade15d87ad.png">  
对比一下列存储的压缩实力：citus里用Columnar引擎存储，不开启压缩的情况下，1200w的数据表size也只有188M  
<img width="409" alt="image" src="https://user-images.githubusercontent.com/3870517/154244014-201fb5ce-db75-493e-adc4-e7c9e96e64e4.png">

因此实际业务的Report表特点是：列非常多，需要进行复杂的JOIN表，每次查询检索的列个数不超过总数的10%，表数据是明显的稀疏矩阵分布。
看到这里大家应该都会想到这样类型的数据结构，简直完整的对标了列存储引擎的优势：
1. 列压缩可以有效解决稀疏矩阵大量数据为0的存储空间问题
2. 每一列都自带索引，任意指标进行ORDER BY都可以获得索引加持

实际上目前使用MySQL行存储是完全能够胜任的，但是秉着优化（搞事）的目的，于是我们进行了复盘和再选型。

## 再选型

这次我们将OLAP型数据库也纳入进考量范畴，待测试的数据库主要有：
* PostgreSQL，算是HTAP数据库，可以加citus_fdw扩展Columnar列存储引擎
* Citus，除了提供分布式，单机下跟PostgreSQL+citus_fdw差不多
* MySQL 8 JSON存储，需要重新验证上文提到“猜的”的那部分内容
* Kudu+Impala，主要是因为神策用的这个引擎
* ClickHouse，主要测试MergeTree
* MariaDB ColumnStore，据说是从Infinidb弄过来的列存储引擎

### PostgreSQL/Citus/citus_fdw

主要测试的是Columnar，压缩方面能力很好，可以把一个1200w行数据，行存储大概3G的表压缩到188M  
<img width="762" alt="image" src="https://user-images.githubusercontent.com/3870517/154403513-90edf6d6-9c5b-4e01-a48a-0e64f0cdf3c1.png">

#### 单表查询性能测试

表大约有1200w行数据  
![image](https://user-images.githubusercontent.com/3870517/154432795-4f6f51fe-a28e-45dd-b70b-4982c0e63905.png)

##### Columnar
![image](https://user-images.githubusercontent.com/3870517/154432957-22f58f98-ca64-4070-8b29-ed3e637bcfd8.png)  
order by spend情况下，查询花费10s `20 rows retrieved starting from 1 in 10 s 44 ms (execution: 10 s 20 ms, fetching: 24 ms)`  
去掉order by spend，查询花费6s `20 rows retrieved starting from 1 in 6 s 367 ms (execution: 6 s 285 ms, fetching: 82 ms)`

##### row-ori
因为spend没有索引，order by spend会导致全表扫描  
![image](https://user-images.githubusercontent.com/3870517/154433347-df5a43db-6a64-4ff1-b918-f5f20ee99b90.png)  
查询花费14s `20 rows retrieved starting from 1 in 14 s 313 ms (execution: 14 s 295 ms, fetching: 18 ms)`

把 order by spend去掉再explain一次，索引出现了  
![image](https://user-images.githubusercontent.com/3870517/154433596-1788e478-fb48-4a8e-a66e-d8fc497c52ef.png)  
查询花费 54ms `20 rows retrieved starting from 1 in 54 ms (execution: 21 ms, fetching: 33 ms)`

#### JOIN表查询性能测试

我们JOIN一个也是1200w的detail类的表，并且group by一个detail表才有的id  
![image](https://user-images.githubusercontent.com/3870517/154435519-eaddd0be-17c2-4b48-ba83-ef6be0c85549.png)

##### Columnar

order by spend情况下，查询花费 `2000 years later ... stop掉了`  
![image](https://user-images.githubusercontent.com/3870517/154436049-1b17396f-d48b-4f8d-806e-290574682c53.png)

不order by spend情况下，查询花费182ms `20 rows retrieved starting from 1 in 182 ms (execution: 158 ms, fetching: 24 ms)`  
![image](https://user-images.githubusercontent.com/3870517/154436825-614f56c9-bc54-411d-a3f2-02d969538c57.png)

测试数据是放在机械硬盘的所以整体表现都比较差，测试过程不是很严谨不过也能说明问题。  
最终没有选择 Citus 系列的存储主要是因为不支持Update和Delete，没有留出任何维修数据的接口。  
PS：PostgreSQL作为一款号称HTAP的数据库，还可以实现OLAP和OLTP分区，比如说当月的数据存在OLTP分区里，频繁Update，T-1的数据存在OLAP分区里压缩永久存储，听起来很不错。

#### MariaDB ColumnStore

并没有进行测试，MariaDB的Columnar没有办法设置排重的Key，直接放弃了，其实硬要用也是可以的，不过看社区也很少，感觉有点坑，过几年再回来看看有没有什么提升

#### Kudu+Impala

并没有进行测试，主要是因为Kudu是Hadoop系的，运维成本比较高，Impala对Golang也比较蛋疼。对于我们这种还要去客户机房部署和运维的SaaS产品，还是要选运维方便点的。不过神策能选用应该还是有些道理，希望有一天能用实际的数据测试一下Kudu

#### ClickHouse 

导入JSON系列的表（原始表字段太多懒得弄了），CH的列压缩能力显现，表的Size都非常小  
<img width="631" alt="QQ20220315-103836@2x" src="https://user-images.githubusercontent.com/3870517/158341793-88a8af36-2e3e-4347-8b8e-d51146fdea53.png">


执行Postgre执行过的SQL，发现性能不咋地，执行时间去到了2.2s  
<img width="711" alt="image" src="https://user-images.githubusercontent.com/3870517/158342544-aa99fad2-c461-4a2e-a990-2aaecf59b2af.png">  
`20 rows retrieved starting from 1 in 2 s 232 ms (execution: 2 s 198 ms, fetching: 34 ms)`

再加上从JSON里解析出数据，再做个ORDER BY，差距不大，CH列即索引的特点在这里还是蛮有优势的，毕竟业务需求上是可以对任意列进行排序  
<img width="710" alt="image" src="https://user-images.githubusercontent.com/3870517/158342855-c231bef0-4008-4910-93d1-8c3cdea4301d.png">  
`20 rows retrieved starting from 1 in 2 s 236 ms (execution: 2 s 186 ms, fetching: 50 ms)`

加上FINAL对性能影响不大，但这里我不太清楚是不是只需要加一个FINAL就可以排所有JOIN表的结果  
<img width="734" alt="image" src="https://user-images.githubusercontent.com/3870517/158343275-142bb8a5-3232-4f2d-be68-e887898f1a3d.png">  
`20 rows retrieved starting from 1 in 2 s 290 ms (execution: 2 s 221 ms, fetching: 69 ms)`

用OLTP的查询思维来查询OLAP可能并不太合适，CH里操作大量JOIN对生产环境并不是太友好，从shell中执行可以看出，process的数据量非常大，从这么大的数据量里捡25条出来，性价比很低  
<img width="814" alt="image" src="https://user-images.githubusercontent.com/3870517/159828909-76559ff2-7d01-4ce7-974c-4255473e3bb0.png">

我们优化一下SQL，先用WHERE约束查询的结果，再进行JOIN  
<img width="805" alt="image" src="https://user-images.githubusercontent.com/3870517/159836937-f6216232-c445-4128-914c-6e584946f9d3.png">  
补上final之后执行，结果提升非常显著，processed的数量明显降低，执行时间也缩短到了0.36s。实际测试如果把final都去掉，执行时间可以降到0.11s  
<img width="808" alt="image" src="https://user-images.githubusercontent.com/3870517/159829357-338168c1-120b-4420-86e7-cd1addb17435.png">  

来看第二个SQL，查另外一个大宽表，不加FINAL的情况下表现非常不错，总耗时只需要500ms，加上FINAL总耗时也只上升到了600ms，成绩优异  
<img width="831" alt="image" src="https://user-images.githubusercontent.com/3870517/158343565-52d19c17-4de7-46f5-aa81-61433f38f38f.png">  
`25 rows retrieved starting from 1 in 517 ms (execution: 472 ms, fetching: 45 ms)`

实测性能很强，完爆其他所有列存储数据库  
最终没有选型的原因是Report当日的数据会频繁更新，而CH的MergeTree排重时间是不可控的，会导致当天的数据产生大量重复，所有SQL都加上FINAL效率未知  
CH始终是标准的OLAP数据库，用来做生产环境页面实时加载报表这样的需求不太Match（这里跟组员观点不同，欢迎讨论）  
另外是实际查询需要JOIN大量MySQL的表，很多表没有办法直接挪到CH上，如果从CH直接查MySQL的表对于SQL要求较高否则会出现全表扫描进CH来运算的问题，比如这样的JOIN表就很容易出现各种奇怪的问题，CH做MySQL的Slave还处于实验室阶段暂时也不考虑  
不过CH依然是非常有利的候选方案，MergeTree一系列的引擎可以说完爆其他OLAP数据库，如果在OLAP和OLTP的联查上能够有更好的突破，或者像PostgreSQL那样提供OLTP和OLAP分区，我还是很乐意选择CH的

#### MySQL 8 JSON存储

主要要补充MySQL 8用JSON存储数据的查询问题。补充一下测试用的表结构大概这样
<img width="729" alt="image" src="https://user-images.githubusercontent.com/3870517/157390480-46e8b355-6972-4386-931c-a9023459ba91.png">
表中有4个JSON字段用于存储指标类的数据，对标测试的使用基本结构的表字段太多就不放出来了。

测试的SQL包含没有索引的ORDER BY和GROUP BY一个外部表的ID，见下图：
<img width="802" alt="image" src="https://user-images.githubusercontent.com/3870517/157402178-539ffe9d-279f-456d-830d-c16b96945dfd.png">

EXPLAIN 执行一下(两个表的EXPALIN执行结果一样就不重复贴了）
<img width="1276" alt="image" src="https://user-images.githubusercontent.com/3870517/157399925-780fe67e-f60a-433a-b1e9-e7fa536ca2d1.png">

#### JSON

实际查询，执行时间在1s左右`25 rows retrieved starting from 1 in 852 ms (execution: 834 ms, fetching: 18 ms)`

#### 原装表

执行时间也是在1s左右`25 rows retrieved starting from 1 in 1 s 173 ms (execution: 1 s 152 ms, fetching: 21 ms)`

#### 结论

用JSON存储，在查询的时候现场JSON_Extract并没有显著降低表的查询速度，如果表列数很多的情况下，用JSON打包反而效率更好一些（测试用的表原装大概有350+个字段），
因此在JSONSchema管理得当的前提下，表用JSON来存储似乎是更优质的选择。

### 最后

附上测试用的表信息（rows的数据不太准确，实际select count(1)显示是一样的行数)
<img width="935" alt="image" src="https://user-images.githubusercontent.com/3870517/157402493-b9808755-1d3d-40da-92db-58535d079528.png">

### 思考

有时候想当然的去认为某件事会有一定翻车的风险，比如说我想当然的认为JSON_Extract会导致性能明显下降，实际看来使用JSON_Extract带来的性能开销并没有我考虑的那么严重

过于严谨的测试也会带来不少损耗，比如说要做这样的测试，从搭建测试环境开始就得花一周以上的时间，如果项目正赶着需要某个功能，而我们又在慢悠悠的做这样的测试，也是不恰当的

用课余时间来进行学术研究，积累经验，等到真正需要使用的时候就可以快速提出具有一定可行性的方法论看起来是可行之道。对于技术工具的使用调研是如此，对于大型项目工程的经验也是如此，技术工具的调研可以课后进行，但是靠谱的项目经验却是更加金贵


