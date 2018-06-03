# 比特币系统的匿名分析

> | Title: |  An Analysis of Anonymity in the Bitcoin System |
> | --- | --- |
> | Authors: | Fergal Reid, Martin Harrigan |
> | Institution: | Clique Reasearch Cluster, Complex and Adaptive Systems Laboratory, University College Dublin, Dublin, Ireland |
> | Journal: | SPSN’ 13 |

## Topic

比特币；交易层；隐私威胁

## Motivation

有没有可能关联同一名用户执行的比特币交易，甚至使用外部信息识别他们？

分析的动机并非去匿名化比特币系统的个体用户，而是证明被动分析公开可用的数据集，使用比特币存在匿名的固有限制。

## Approach

#### 交易网络

交易网络代表交易间的比特币流。每个顶点代表一条交易，每条源点和终点间的有向边代表交易的输出对应源点是交易的输入对应终点。每条有向边也包含比特币值和时间戳。

![An example sub-network from the transaction network. Each rectangular vertex represents a transaction and each directed edge represents a flow of Bitcoins from an output of one transaction to an input of another](https://media.springernature.com/original/springer-static/image/chp%3A10.1007%2F978-1-4614-4139-7_10/MediaObjects/273444_1_En_10_Fig2_HTML.gif)

#### 用户网络

用户网络代表用户间的比特币流。每个顶点代表一名用户。每条源点和终点间的有向边代表一条交易的输入-输出对，其中输入的公钥属于的用户对应源点，输出的公钥属于的用户对应终点。每条有向边也包含比特币值和时间戳。

**预处理步骤**利用特性：在多输入的交易中，链接是不可避免的，必然揭露输入被同一名用户拥有。风险是如果揭露了一个公钥的所有者，链接能够揭露其他属于同一名所有者的交易。

![An example sub-network from the incomplete network. Each diamond vertex represents a public-key and each directed edge between diamond vertices represents a flow of Bitcoins from one public-key to another](https://media.springernature.com/original/springer-static/image/chp%3A10.1007%2F978-1-4614-4139-7_10/MediaObjects/273444_1_En_10_Fig5_HTML.gif)

![An example sub-network from the user network. Each circular vertex represents a user and each directed edge between circular vertices represents a flow of Bitcoins from one user to another. The maximal connected component from the ancillary network that corresponds to the vertex _u_  1 is shown within the _dashed grey box_](https://media.springernature.com/original/springer-static/image/chp%3A10.1007%2F978-1-4614-4139-7_10/MediaObjects/273444_1_En_10_Fig6_HTML.gif)

#### 匿名分析

* **融合离线网络信息**：考虑若干公开课用的数据源，与用户网络融合；
* **TCP/IP层信息**：“第一个告诉你交易的节点可能就是交易的始发节点”，除非用户使用匿名代理技术，有可能映射公钥和IP地址；
* **个人人际分析和可视化**：对于任意参与用户，可以直接从用户网络中获得信息片段；

![An egocentric visualization of the vertex representing the WikiLeaks public-key in the incomplete user network. The size of a vertex corresponds to its degree in the entire incomplete user network. The _color_ denotes the volume of Bitcoins – lighter colors have larger volumes flowing through them](https://media.springernature.com/original/springer-static/image/chp%3A10.1007%2F978-1-4614-4139-7_10/MediaObjects/273444_1_En_10_Fig9_HTML.gif)

* **上下文发现**：给定若干有兴趣的公钥或用户，使用网络结构和上下文更好地理解他们之间的比特币网络流；

![A visualisation of all users identified and all shortest paths between the _vertices_ representing those users and the _vertex_ representing the MyBitcoin service in the user network](https://media.springernature.com/original/springer-static/image/chp%3A10.1007%2F978-1-4614-4139-7_10/MediaObjects/273444_1_En_10_Fig10_HTML.gif)

* **流和时序分析**：跟踪网络上大量的值流动；

![Bitcoins are transferred very quickly, between the public-keys on the highlighted paths](https://media.springernature.com/original/springer-static/image/chp%3A10.1007%2F978-1-4614-4139-7_10/MediaObjects/273444_1_En_10_Fig14_HTML.gif)

## Contribution

针对维基解密公布的账户进行数据分析，能够统计出维基解密网站公布的比特币地址的资金余额、资金来源和资金流向。

针对比特币中公开的一个盗窃地址最密切的5个地址，揭示攻击者盗窃前的行为和盗窃后的资金流向。

## Performance

#### Dataset

对比社交网络数据集，有两个值得关注的特征：

* 公开和私有的描述是清楚的：比特币交易的全部历史是公开可用的。
* 比特币系统没有使用政策。在加入比特币的对等网络后，客户端能够自由地请求比特币交易的全部历史；无需使用爬虫拼凑。

#### Baseline

/

#### Metric

![](https://media.springernature.com/original/springer-static/image/chp%3A10.1007%2F978-1-4614-4139-7_10/MediaObjects/273444_1_En_10_Fig3_HTML.gif)

交易网络图a展示了双对数坐标系中的累积度数分布：实红曲线是累积入度和出度分布；虚绿曲线是累积入度分布；点蓝曲线是累积出度分布。图b展示了双对数坐标系中的累积构成分布。图c-e分别展示了每月交易网络的边数、密度和平均路径长度。

**用户网络**

![User network. (**a**) Log–log plot of the cumulative degree distributions. (**b**) Log–log plot of the cumulative component size distribution. (**c**) Temporal histogram showing the number of edges per month. (**d**) Temporal histogram showing the density per month. (**e**) Temporal histogram showing the average path length per month](https://media.springernature.com/original/springer-static/image/chp%3A10.1007%2F978-1-4614-4139-7_10/MediaObjects/273444_1_En_10_Fig4_HTML.gif)

用户网络图a展示了双对数坐标系中的累积度数分布。图c-e分别展示了每月用户网络的边数、密度和平均路径长度。
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIxMzQyOTkxMDZdfQ==
-->