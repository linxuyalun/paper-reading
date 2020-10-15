* [流计算精品翻译: The Dataflow Model](https://developer.aliyun.com/article/64911)
* [《The Dataflow Model》论文翻译](https://apparition957.github.io/2020/01/07/%E3%80%8AThe-Dataflow-Model%E3%80%8B%E8%AE%BA%E6%96%87%E7%BF%BB%E8%AF%91/)

## Introduction

实践表明，我们**永远无法同时优化数据处理的准确性、延迟程度和处理成本等各个维度**。因此，数据工作者面临如何协调这些几乎相互冲突的数据处理技术指标的窘境，设计出来各种纷繁的数据处理系统和实践方法。

考虑一个例子：一家流媒体平台提供商通过视频广告，向广告商收费把视频内容进行商业变现。收费标准按广告收看次数、时长来计费。这家流媒体的平台支持在线和离线播放。**流媒体平台提供商希望知道每天向广告商收费的金额，希望按视频和广告进行汇总统计。**另外，他们想在大量的历史离线数据上进行历史数据分析，进行各种实验。

<u>广告商和内容提供者想知道视频被观看了多少次，观看了多长时间，视频被播放时投放了哪个广告，或者广告播放是投放在哪个视频内容中，观看的人群统计分布是什么。</u>

**广告商也很想知道需要付多少钱，而内容提供者想知道赚到了多少钱。 而他们需要尽快得到这些信息，以便调整预算/调整报价，改变受众，修正促销方案，调整未来方向**。所有这些越实时越好，因涉及到金额，准确性是至关重要的。

尽管数据处理系统天生就是复杂的，视频平台还是希望一个简单而灵活的编程模型。最后，由于他们基于互联网的业务遍布全球，他们需要的系统要能够处理分散在全球的数据。

上述场景需要计算的指标包括**每个视频观看的时间和时长，观看者、视频内容和广告是如何组合的**（即按用户，按视频的观看“会话”）。概念上这些指标都非常直观，但是现有的模型和系统并无法完美地满足上述的技术要求。