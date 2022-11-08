# ClickHouse



## MergeTree引擎

![干货连载 | ClickHouse内核分析-MergeTree的存储结构和查询加速](https://picx1.zhimg.com/v2-3a90cbc570805ab69ba7fe57de5b6079_720w.jpg?source=172ae18b)

### MergeTree思想

提到MergeTree这个词，可能大家都会联想到LSM-Tree这个数据结构，我们常用它来解决随机写磁盘的性能问题，MergeTree的核心思想和LSM-Tree相同。MergeTree存储结构需要对用户写入的数据做排序然后进行有序存储，数据有序存储带来两大核心优势：

• 列存文件在按块做压缩时，排序键中的列值是连续或者重复的，使得列存块的数据压缩可以获得极致的压缩比。

• 存储有序性本身就是一种可以加速查询的索引结构，根据排序键中列的等值条件或者range条件我们可以快速找到目标行所在的近似位置区间(下文会展开详细介绍)，而且这种索引结构是不会产生额外存储开销的。

### MergeTree简介

![img](https://p5.itc.cn/q_70/images03/20210401/c9a0471a49224a9282a34954e7546d18.png)

MergeTree在写入一批数据时，数据总会以数据片段的形式写入磁盘，且数据片段不可修改。为了避免片段过多，ClickHouse会通过后台线程，定期合并这些数据片段，属于相同分区的数据片段会被合成一个新的片段。这种数据片段往复合并的特点，也正是合并树名称的由来。

包括基础的MergeTree，拥有数据去重能力的ReplacingMergeTree、CollapsingMergeTree、VersionedCollapsingMergeTree，拥有数据聚合能力的SummingMergeTree、AggregatingMergeTree等。但这些拥有“特殊能力”的MergeTree表引擎在存储上和基础的MergeTree其实没有任何差异，它们都是在数据Merge的过程中加入了“额外的合并逻辑”

### MergeTree的存储结构

![图片](http://inews.gtimg.com/newsapp_bt/0/14728809327/641)

（1）partition：分区目录，余下各类数据文件（primary.idx、[Column].mrk、[Column]. bin等）都是以分区目录的形式被组织存放的，属于相同分区的数据，最终会被合并到同一个分区目录，而不同分区的数据，永远不会被合并在一起。更多关于数据分区的细节会在6.2节阐述。

（2）checksums.txt：校验文件，使用二进制格式存储。它保存了余下各类文件(primary. idx、count.txt等)的size大小及size的哈希值，用于快速校验文件的完整性和正确性。

（3）columns.txt：列信息文件，使用明文格式存储。用于保存此数据分区下的列字段信息，例如：

（4）count.txt：计数文件，使用明文格式存储。用于记录当前数据分区目录下数据的总行数：

（5）primary.idx：一级索引文件，使用二进制格式存储。用于存放稀疏索引，一张MergeTree表只能声明一次一级索引（通过ORDER BY或者PRIMARY KEY）。借助稀疏索引，在数据查询的时能够排除主键条件范围之外的数据文件，从而有效减少数据扫描范围，加速查询速度。

（6）[Column].bin：数据文件，使用压缩格式存储，默认为LZ4压缩格式，用于存储某一列的数据。由于MergeTree采用列式存储，所以每一个列字段都拥有独立的．bin数据文件，并以列字段名称命名（例如CounterID.bin、EventDate.bin等）。

（7）[Column].mrk：列字段标记文件，使用二进制格式存储。标记文件中保存了．bin文件中数据的偏移量信息。标记文件与稀疏索引对齐，又与．bin文件一一对应，所以MergeTree通过标记文件建立了primary.idx稀疏索引与．bin数据文件之间的映射关系。即首先通过稀疏索引（primary.idx）找到对应数据的偏移量信息（.mrk），再通过偏移量直接从．bin文件中读取数据。由于．mrk标记文件与．bin文件一一对应，所以MergeTree中的每个列字段都会拥有与其对应的．mrk标记文件（例如CounterID.mrk、EventDate.mrk等）。

（8）[Column].mrk2：如果使用了自适应大小的索引间隔，则标记文件会以．mrk2命名。它的工作原理和作用与．mrk标记文件相同。

（9）partition.dat与minmax_[Column].idx：如果使用了分区键，例如PARTITION BY EventTime，则会额外生成partition.dat与minmax索引文件，它们均使用二进制格式存储。partition.dat用于保存当前分区下分区表达式最终生成的值；而minmax索引用于记录当前分区下分区字段对应原始数据的最小和最大值。例如EventTime字段对应的原始数据为2019-05-01、2019-05-05，分区表达式为PARTITION BY toYYYYMM(EventTime)。partition.dat中保存的值将会是2019-05，而minmax索引中保存的值将会是2019-05-012019-05-05。

在这些分区索引的作用下，进行数据查询时能够快速跳过不必要的数据分区目录，从而减少最终需要扫描的数据范围。

（10）skp_idx_[Column].idx与skp_idx_[Column].mrk：如果在建表语句中声明了二级索引，则会额外生成相应的二级索引与标记文件，它们同样也使用二进制存储。二级索引在ClickHouse中又称跳数索引，目前拥有minmax、set、ngrambf_v1和tokenbf_v1四种类型。这些索引的最终目标与一级稀疏索引相同，都是为了进一步减少所需扫描的数据范围，以加速整个查询过程。