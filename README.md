# best practice with Spark Core and Spark SQL

## 用DataFrame替代RDD
RDD是Spark Core的核心抽象，而DataFrame是Spark SQL的核心抽象。从性能角度，DataFrame相比RDD有巨大优势，主要体现在：
>* 支持谓词下推。对logical plan（即DAG）重排序，把过滤操作推到尽可能靠近数据源的地方，从而让下游的操作在尽可能小的数据集上进行。
>* 处理每条数据时，不需要针对完整的scala object，而只需要针对必要的field，从而大幅减少需要反序列化以及网络传输的数据量。
>* 更快、更省内存的序列化和反序列化方式
>* 数据的列式存储，更高的存储效率，更少的IO
>* 支持off-heap的内存管理，从而使应用程序免受GC造成的停顿和时延
为什么DataFrame会有性能提升？因为相较RDD而言，DataFrame对数据和操作有更深入的洞察：
>* DataFrame支持的数据类型都是预定义好的数据类型，数据以结构化形式存储，DataFrame知道数据的schema，可以对数据的存储和操作进行优化。而RDD中，数据的存储结构是不透明的，对象是不透明的、任意的，无法做假设，也无法做优化。
>* DataFrame支持的操作都是预定义好的高度优化过的关系型操作（如Select, Order by, Filter, Group by等等），而RDD的操作是可任意定义的functional操作。functional操作是不透明的，无法做优化。
DataFrame是如何进行优化的？答案是通过Catalyst和Tungsten：
>* Catalyst Query Optimizer。Catalyst负责把Spark SQL代码编译成RDD代码。在这个过程中，由于Catalyst知道数据的schema，也了解每一种关系型操作，因此可以对计算过程进行优化。例如：对logical plan（即DAG）进行重排序，把filter尽可能往前推；对于每一行数据（即一个serialized scala object），只select, deserialize必要的column（即object的一个field），而不需要deserialize整个object再进行操作；等等
>* Tungsten Off-heap Serializer。由于DataFrame的数据类型都是预先限定好的，且数据的schema已知，因此Tungsten可以充分利用这些信息，高效的实现数据的序列化和反序列化。无论是内存占用还是时间消耗，Tungsten的序列化和反序列化都大大优于java原生的Serializer以及Kryo。此外，Tungsten是列式存储的，而数据处理中大部分情况都是基于列进行选择、聚合或排序，所以列式存储的设计可以减少IO、提高处理效率、提高数据压缩效率。最后，Tungsten支持off-heap的方式来管理内存，从而免受GC的影响。
DataFrame的限制或代价
>* 数据必须要有schema，如果数据本身是非结构化的（例如图片，文本），很难提取schema，那么没法用DataFrame
>* 数据必须表达成Spark SQL预定义的数据类型。否则Tungsten不会工作
>* DataFrame支持的操作不像RDD那么丰富和底层。例如RDD有丰富的API可以方便的控制partitioning，而DataFrame则没有。

## 减少shuffle
shuffle意味着网络传输，而网络传输意味着高时延。通过尽可能的减少shuffle，把时间花在计算而非网络传输，可以提高Spark应用的执行效率和速度。减少shuffle有多种方式：
1. 采用mapper side reduce来减少shuffle。例如用reduceByKey代替groupByKey+reduce，这样在shuffle之前先做一轮reduce，可以大幅减少需要shuffle的数据量；
2. 通过Pre-partition来减少shuffle。例如一种情况，假设需要周期性的对两个RDD进行join，其中一个RDD是静态的、不随时间变化的（例如用户注册信息），另一个RDD是动态的、时变的（例如用户在某个时间段内的活动），那么在join之前先对静态RDD进行pre-partition，这样每次join时，静态RDD的partitioner已知，只有时变RDD会发生shuffle。
3. 通过broadcast来减少shuffle。例如一大一小两个RDD进行join，那么可以先把小的RDD collect到driver上形成一个查找表，然后把这个查找表作为广播变量传播到各个executor上，然后对大的RDD进行mapPartitions，每个partition跟查找表做local combine。这样可以达到join的效果并完全避免了shuffle。这种join有个专门的名字叫“broadcast hash join”，事实上，Spark SQL支持自动进行broadcast hash join，而Spark Core则需要手动去写代码实现

## 正确使用persist（缓存）
缓存的意义在于避免数据的重复计算。假如某一个RDD会在两个不同的action中被用到，那么把这个RDD缓存起来可以避免重复计算（即当第一个action计算的过程中得到了这个RDD的时候，这个RDD会缓存在spark的内存或磁盘上，当第二个action需要用到这个RDD的时候，可以直接读缓存，而不必重新计算）。一个需要用到缓存的场景是对RDD进行自定义partition的时候，如果重新partition之后的RDD需要重复使用，那么需要在partitionBy之后缓存起来，否则每次使用时都会导致RDD重新partition，而partition是非常昂贵的操作（因为涉及shuffle）。
初学者常常犯的错误并不是在需要缓存的时候忘记缓存，而是在不需要缓存的时候却进行了缓存。他们潜意识里认为把RDD缓存起来对fault tolerance有帮助，但其实缓存并不是fault tolerance的手段，Spark Core的fault tolerance是通过DAG的lineage来实现的。不必要的缓存只是单纯的浪费了内存资源，对计算并没有任何帮助。

## 使用广播变量
广播变量是Spark中的一种数据分发方式，可以高效的把driver中的对象分发到各个executor。假如某个transformation的lambda表达式中用到了一个查找表，如果采用普通的方式，这个查找表会随着lambda表达式复制到每一个相关的task中，如果有1000个task就会复制1000份，这既浪费了driver的传输带宽，也浪费了executor的内存资源；如果采用广播变量的方式，那么查找表会通过高效的传播协议（多点到多点，类似BitTorrent）来传播，并且每个executor只复制一份，同一个executor中的不同tasks共享这一份数据。

## 处理数据倾斜
Spark的job以shuffle为界划分成一个个stage，stage之间串行运行，每个stage都要等前一个stage结束才能开始运行。每个stage包含多个并行处理单元也就是task，不同的task在不同的数据子集上执行相同的处理逻辑，一个stage的运行时间取决于这个stage里运行最久的那个task的运行耗时。理想情况下，同一个stage的不同的task分配到的数据量规模和运行耗时应该大体接近，但实际中往往不是如此。例如有时候一个stage的大部分task都只花了几秒或十几秒，但是有个别task却花了几十分钟，严重拖慢了stage进度，这种情况通常就是发生了数据倾斜。 
数据倾斜也就是数据在不同task之间划分的不均匀。在基于key的aggregation或join操作中，如果某些key出现的次数明显多于别的key，由于同一个key的数据一定会拉取到同一个task中处理，那么这个倒霉的task就需要处理比别的task多得多的数据，从而成为整个stage的瓶颈。
对于aggregation中的数据倾斜，通常的应对思路是把集中的key进行打散，通过二次聚合的方式，把高负荷task的数据分散给更多的task并行处理。以一个实际的场景为例：现在有一个用户购买记录的数据集，假设每条record包含的字段有userId、购买的产品信息、以及支付金额，我们需要统计每个用户的总消费金额。数据存在严重的倾斜，表现在大部分userId只有一两条购买记录，但是小部分高频用户有上万条记录。我们可以通过一些手段（数据量小的话可以通过reduceByKey统计每个userId出现的次数，然后sort并取最高频的N个userId，数据量大的话可以先sample）找到这些高频用户，把高频用户单独作为一个RDD特殊处理，普通用户作为另一个RDD按常规方式处理（两个RDD处理结果union起来就是最终结果）；在高频用户RDD中，每个userId加上一个取值为1~n的随机前缀，假设有一个高频userId“Alice”出现了很多次，加上随机前缀之后就会变成”1_Alice”，”2_Alice”，…,“n_Alice”，这样的话原来必须由同一个task聚合的”Alice”就可以分给多个task聚合，最后只需要把n个”*_Alice”二次聚合，就可以得到“Alice”总的聚合结果。
对于join中的数据倾斜，可以采用broadcast hash join的方式来解决。假设要对两个RDD进行join，其中一个RDD（称其为rdd1）里存在着一些高频key，那么首先找到这些高频key，然后对另一个RDD(称其为rdd2）进行filter和collect，把高频key对应的数据以HashMap的形式保存下来，并把这个hashMap作为广播变量传播到每一个executor上。接下来，rdd1通过hashMap进行过滤并分成两部分：一部分是高频key对应的数据，这部分跟hashMap通过local combine生成join结果；另一部分是正常key对应的数据，这部分可以直接跟rdd2进行join。两部分的结果union起来就是最终结果。

## 用foreachPartitions代替foreach

## 用flatMap代替map+filter。
map+filter需要遍历数据两次，而flatMap可以实现同样的功能，但是只需遍历数据一次。（不过随着Spark Core的进化，map+filter这种窄依赖操作可以合并到一个stage内，也就是说只需遍历一次就能完成map+filter。即便如此， 用flatMap代码也更简洁一些）
