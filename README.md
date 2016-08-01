# 融合突变检测程序（fuseek）

## 简介

本程序针对DNA样本的二代测序数据（主要指目标区域捕获测序，且为双端测序），基于bwa mapping得到的BAM文件，从中检测基因融合突变（fusion）。

为方便字符串解析和复杂数据结构的灵活使用，本程序最终采用perl语言编写，以单个命令（'fuseek'）的形式发布。每个分析步骤都分别封装为一个子命令（如'collect'、'identify'等），并最终组合封装成一个便于整体调用的子命令'run'。


## 安装说明

本程序只包含单个程序文件，几乎不用安装，直接拷贝，在安装有perl的环境中，即可使用。

在Linux环境中，为方便使用，可以将程序放入`/usr/bin/`或其它被加入PATH环境变量的目录中，并将其加上可执行的属性：

	$ chmod +x /path/to/fuseek

如果有在运行本程序时，出现如下错误提示：

	Can't locate List/BinarySearch.pm in @INC (you may need to install the List::BinarySearch module) (@INC contains: ... )

则可使用cpan安装相应的perl模块：

	$ cpan
	cpan> install List::BinarySearch


## 使用方法

对于某个已经完成bwa mapping的BAM文件：

	$ fuseek run hg19.fa refGene.gz in.bam foo

将生成如下两个文件：

	foo.1-discord.txt
	foo.2-breakpoints.txt
	foo.3-fusions.txt
	foo.4-fusions.sorted.txt
	foo.5-fusions.sorted.annotated.txt
	foo.6-fusions.sorted.annotated.filtered.txt

该流程也可以拆分为如下几个步骤分开顺序执行：

	$ fuseek collect in.bam foo.1-discord.txt
	$ fuseek identify hg19.fa foo.1-discord.txt foo.2-breakpoints.txt
	$ fuseek refine foo.2-breakpoints.txt in.bam foo.3-fusions.txt
	$ fuseek sort foo.3-fusions.txt foo.4-fusions.sorted.txt
	$ fuseek annotate hg19.refGene.gz foo.4-fusions.sorted.txt foo.5-fusions.sorted.annotated.txt
	$ fuseek filter foo.5-fusions.sorted.annotated.txt foo.6-fusions.sorted.annotated.filtered.txt

## 分析思路步骤及各子命令简介

1. 'collect' - 从BAM文件中提取双端比对位置不一致的序列。

	在这一步骤中，只保留双端都能够唯一比对上（但是可能存在soft clip）的read pairs，选取其中双端距离不符合预期（非同一染色体、或距离过远，比如超过1000kb），并在结果文件中，将相应的序列、质量等信息都保存下来，以备使用。由于这类reads本身比例就不大，所以即使将其在perl程序中全部读入内存再处理，也都还是可以接受的。

2. 'identify' - 根据比对位置不一致的序列，结合其比对cigar中的soft clip信息，从中识别出可能的融合突变及其断点。

	从位置不一致的序列对中，根据其cigar等信息（soft clip），确定可能的断点。这里，只保留那些在read的3'端出现soft clipping的情况，并预期被clip掉的部分，应该与pair中另一条read能够比对到比较接近的位置。将此clip序列，放到pair read所比对的位置附近（上游100bp，下游1000bp）再次进行比对。若能成功比对，则保留此soft clip位置推断得到的断点，并将此pair read作为其支持证据留下。

3. 'refine' - 在断点附近重新做比对，优化结果，同时也确定各融合突变的频率。

	这里仅根据所报出的各断点位置，对fusion进行合并；然后回到bam文件中，统计断点处的测序深度，用以估算fusion发生的频率。

4. 'sort' - 按照基因组坐标进行排序

5. 'annotate' - 对融合突变进行注释，确定其基因及其外显子等信息。

	对fusion进行注释，这里可能会同时得到多个转录本信息，于是会将其都列出（用半角分号隔开）。

6. 'filter' - 过滤：要求同时落在基因区，且两端在不同基因，且支持reads数目超过1。
