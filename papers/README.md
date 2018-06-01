# 区块链论文

区块链的前沿论文

## 目录

* [Survey](#survey)
* [Consensus](#consensus)
* [Attacks](#attacks)
* [Data Mining](#data-mining)
* [Privacy](#privacy)
* [Applications](#applications)

---

### Survey

| Title | Authors | Published in | Year | Files | Notes | Supplementaries |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| Making Sense of Blockchain Applications: A Typology for HCI | Chris Elsden, Arthi Manohar, Jo Briggs, Mike Harding, Chris Speed, John Vines | CHI | 2018 | [:ledger:](https://dl.acm.org/ft_gateway.cfm?id=3174032&type=pdf) |  | [:floppy_disk:](https://figshare.com/articles/Survey_and_Typology_of_Blockchain_Applications_Sep_2017_/5765502/1) |

### Consensus

| Title | Authors | Published in | Year | Files | Notes | Supplementaries |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| Thunderella: Blockchains with Optimistic Instant Confirmation | Rafael Pass, Elaine Shi | EUROCRYPT | 2018 |  |  |
| But Why Does It Work? A Rational Protocol Design Treatment of Bitcoin | Christian Badertscher, Juan A. Garay, Ueli Maurer, Daniel Tschudi, Vassilis Zikas | EUROCRYPT | 2018 |  |  |
| Ouroboros Praos: An Adaptively-Secure, Semi-synchronous Proof-of-Stake Blockchain | Bernardo David, Peter Gazi, Aggelos Kiayias, Alexander Russell | EUROCRYPT | 2018 |  |  |
| Sustained Space Complexity | Joël Alwen, Jeremiah Blocki, Krzysztof Pietrzak | EUROCRYPT | 2018 |  |  |
| Simple Proofs of Sequential Work | Bram Cohen, Krzysztof Pietrzak | EUROCRYPT | 2018 |  |  |
| Ouroboros: A Provably Secure Proof-of-Stake Blockchain Protocol | Aggelos Kiayias, Alexander Russell, Bernardo David, Roman Oliynykov | CRYPTO | 2017 |  | [:memo:](./notes/Ouroboros.md) | [:keyboard:](https://github.com/input-output-hk/cardano-sl.git) |
| Algorand: Scaling Byzantine Agreements for Cryptocurrencies | Yossi Gilad, Rotem Hemo, Silvio Micali, Georgios Vlachos, Nickolai Zeldovich | SOSP | 2017 | [:ledger:](https://dl.acm.org/ft_gateway.cfm?id=3132757&type=pdf) | [:memo:](./notes/Algorand演示文档.pdf) |
| Secure High-Rate Transaction Processing in Bitcoin | Yonatan Sompolinsky, Aviv Zohar | Financial Cryptography | 2013 |[:ledger:](https://fc15.ifca.ai/preproceedings/paper_30.pdf)   |  [:memo:](./notes/GHOST阅读笔记.pdf) |

### Attacks

| Title | Authors | Published in | Year | Files | Notes | Supplementaries |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| A Survey of Attacks on Ethereum Smart Contracts (SoK) | Nicola Atzei, Massimo Bartoletti, Tiziana Cimoli | POST | 2017 | [:ledger:](https://pdfs.semanticscholar.org/66cc/6e3f36c4282a189249523a5e88577739b736.pdf) |  |

### Data Mining

| Title | Authors | Published in | Year | Files | Notes | Supplementaries |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| Detecting Ponzi Schemes on Ethereum: Towards Healthier Blockchain Technology|Weili Chen, Weili Chen, Zibin Zheng, Jiahui Cui, Edith Ngai, Peilin Zheng,  Yuren Zhou| WWW | 2018 | [:ledger:](https://dl.acm.org/citation.cfm?id=3186046)|[:memo:](./notes/Detecting%20Ponzi%20Schemes%20on%20Ethereum.pdf)|                                                                               
| Dissecting Ponzi schemes on Ethereum: identification, analysis, and impact|Massimo Bartoletti, Salvatore Carta, Tiziana Cimoli, Roberto Saia | | 2017 | [:ledger:](http://pdfs.semanticscholar.org/96ae/cce3b5f82a6e920445eb4bfd8ab71189538a.pdf)|[:memo:](./notes/Dissecting%20Ponzi%20schemes%20on%20Ethereum.pdf)|                                                                               
| Data mining for detecting Bitcoin Ponzi schemes|Massimo Bartoletti, Barbara Pes, Sergio Serusi | | 2017 | |[:memo:](./notes/Data%20mining%20for%20detecting%20Bitcoin%20Ponzi%20schemes.pdf)|                                                                               


### Privacy

> 大多数区块链的设计是为了保护交易的完整性，但他们不考虑交易隐私。一个区块链系统具有交易隐私是指当交易关联信息被保护，且只有参与者知道交易内容。一方面，在私有设置中，交易历史的完全透明性可能不是问题。透明性对于私有系统来说是需要的（例如财务审计），可以通过直接添加访问控制层来保护区块链数据。另一方面，在公共环境中，交易隐私的需求是由两个因素驱动的。首先，去匿名化攻击可以成功地复原出比特币网络的底层结构，甚至将比特币地址与现实世界的身份联系起来。其次，交易的可链接性会破坏货币的互换性，通过历史数据挖掘分析，可以使一些电子货币比其他电子货币更有价值。——Pro. Ooi

| Title | Authors | Published in | Year | Files | Notes | Supplementaries |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| 区块链隐私保护研究综述 | 祝烈煌, 高峰, 沈蒙, 李艳东, 郑宝昆, 毛洪亮, 吴震 | 计算机研究与发展 | 2017 | [:ledger:](http://crad.ict.ac.cn/CN/article/downloadArticleFile.do?attachType=PDF&id=3529) | [:memo:](./notes/区块链隐私保护研究综述.md) |
| Deanonymisation of Clients in Bitcoin P2P Network | Alex Biryukov, Dmitry Khovratovich, Ivan Pustogarov | CCS | 2014 | [:ledger:](https://www.cryptolux.org/images/a/a1/Ccsfp614s-biryukovATS.pdf) | [:memo:](./notes/Deanonymisation%20of%20Clients%20in%20Bitcoin%20P2P%20Network.pdf) | [:camera:](http://www.hiero.lu/doc/H1_Alex-Biryukov-Research-CC.pdf) |

### Applications

| Title | Authors | Published in | Year | Files | Notes | Supplementaries |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| BigchainDB: a scalable blockchain database | McConaghy, Trent, Rodolphe Marques, Andreas Müller, Dimitri De Jonghe, Troy McConaghy, Greg McMullen, Ryan Henderson, Sylvain Bellemare, and Alberto Granzotto |  | 2016 | [:ledger:](https://www.bigchaindb.com/whitepaper/bigchaindb-whitepaper.pdf) |  |

> Legend:
> 
> | :ledger: | :memo: | :keyboard: | :camera: | :floppy_disk: |
> | :-: | :-: |  :-: |  :-: |  :-: | 
> | PDF<br>Files | Notes | Code | Slides | Other<br>Supplementaries |


