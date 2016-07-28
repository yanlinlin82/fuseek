# 融合突变检测

## 简介

本程序针对样本DNA的二代测序数据（主要指目标区域捕获测序，且为双端测序），基于bwa mapping得到的BAM文件，从中检测基因融合突变（fusion）。

为方便字符串解析和复杂数据结构的灵活使用，本程序最终采用perl语言编写，以单个命令（'fuseek'）的形式发布。每个分析步骤都分别封装为一个子命令（如'collect'、'identify'等），并最终包装一个便于整体调用的命令'run'。

## 分析思路步骤

1. 'collect' - 从BAM文件中提取双端比对位置不一致的序列。

2. 'identify' - 根据比对位置不一致的序列，结合其比对cigar中的soft clip信息，从中识别出可能的融合突变及其断点。

3. 'refine' - 在断点附近重新做比对，优化结果，同时也确定各融合突变的频率。

4. 'annotate' - 对融合突变进行注释，确定其基因及其外显子等信息。

## 安装说明

本程序对第三方程序的依赖性极少，一般而言只要安装了perl即可运行。

如果有在运行本程序时，出现如下错误提示：

	Can't locate List/BinarySearch.pm in @INC (you may need to install the List::BinarySearch module) (@INC contains: ... )

则可使用cpan安装相应的perl模块：

	$ cpan
	cpan> install List::BinarySearch
