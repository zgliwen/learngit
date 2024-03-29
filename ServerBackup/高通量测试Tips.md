### **用MD5算法验证网络文件传输的完整性**

	MD5 (Message-Digest Algorithm 5), 对任意长度的信息逐位进行计算，产生一个二进制长度为128位（十六进制长度就是32位）的“指纹”（或称“报文摘要”），不同文件产生相同的报文摘要的可能性非常小。

#### 1. 产生指纹

	md5sum ./* > file.md5

> 指纹的保存文件无需以md5为后缀，这只是一个记录各个文件指纹的文本文件

#### 2. 验证文件完整性

	md5sum -c file.md5


> 对file.md5文件中的待验证文件列表依次检测，如果验证无误，输出类似"文件名：确定"的信息

### **关注测序数据**

> **read各个位置的碱基质量值分布** 
	
	要求各碱基质量值高，波动小（质量稳定）

> **碱基的总体质量值分布** 

	对于二代测序，最好是达到Q20的碱基在95%以上（最差不低于90%），Q30要求大于85%（最差不低于80%）。

> **read各个位置上碱基分布比例，目的是为了分析碱基的分离程度**

	如果测序过程随机，那么每个位置上A和T的比例应该差不多，C和G的比例应该差不多，偏差最好平均在1%以内。如果过高，除非有合理的原因，比如某些特定的捕获测序所致，否则都需注意是不是测序过程有什么偏差。

> **GC含量分布，用于协助判断测序过程是否随机**

	对于人类来说，基因组GC含量在40%左右，如果发现GC含量的图谱明显偏离这个值，说明测序过程中存在较高的序列偏向性，结果就是基因组中某些特定区域被反复测序的几率高于平均水平。覆盖度偏离，会影响下游的变异检测和CNV分析。

> **read各位置的N含量**

	理论上，在测序数据中不应该出现N，如果出现，说明测序的光学信号无法被清晰分辨，如果这种情况较多，说明测序系统或者测序试剂的错误。

> **read是否包含测序的接头序列**

	当测序read的长度大于被测序的DNA片段（即插入片段）时，会在read的末尾测到这些接头序列。一般的WGS测序不会测到接头序列，因为构建WGS测序的文库序列（插入片段）都比较长，约几百bp，而read的测序长度在100-150 bp。但是在进行一些RNA测序时，由于它们的序列本身较短，很多只有几十bp（特别是miRNA），那么很容易会出现read测通的现象，这个时候就会在read的末尾测到这些接头序列。

> **read重复率，这是实验扩增过程所引入的**

查看测序质量

	fastqc input.fq -o fastqc_out_dir/
	#fastqc_out_dir需要提前构建

去除低质量reads Trimmomatic，若reads中有接头，可以同时指定接头文件

PE 

	$ java -jar /path/Trimmomatic/trimmomatic-0.36.jar PE -phred33 -trimlog logfile reads_1.fq.gz reads_2.fq.gz out.read_1.fq.gz out.trim.read_1.fq.gz out.read_2.fq.gz out.trim.read_2.fq.gz ILLUMINACLIP:/path/Trimmomatic/adapters/TruSeq3-PE.fa:2:30:10 SLIDINGWINDOW:5:20 LEADING:5 TRAILING:5 MINLEN:50

SE

	$ java -jar /path/Trimmomatic/trimmomatic-0.36.jar SE -phred33 -trimlog se.logfile raw_data/untreated.fq out.untreated.fq.gz ILLUMINACLIP:/path/Trimmomatic/adapters/TruSeq3-SE.fa:2:30:10 SLIDINGWINDOW:5:20 LEADING:5 TRAILING:5 MINLEN:50

ILLUMINACLIP，接头序列切除参数。LLUMINACLIP:TruSeq3-PE.fa:2:30:10（省掉了路径）意思分别是：TruSeq3-PE.fa是接头序列，2是比对时接头序列时所允许的最大错配数；30指的是要求PE的两条read同时和PE的adapter序列比对，匹配度加起来超30%，那么就认为这对PE的read含有adapter，并在对应的位置需要进行切除【注】。10和前面的30不同，它指的是，我就什么也不管，反正只要这条read的某部分和adpater序列有超过10%的匹配率，那么就代表含有adapter了，需要进行去除；

> 【注】测序的时候一般只会测到一部分的adapter，因此read和adaper对比的时候肯定是不需要要求百分百匹配率的，上述30%和10%其实是比较推荐的值。
SLIDINGWINDOW，滑动窗口长度的参数，SLIDINGWINDOW:5:20代表窗口长度为5，窗口中的平均质量值至少为20，否则会开始切除；


LEADING，规定read开头的碱基是否要被切除的质量阈值；
TRAILING，规定read末尾的碱基是否要被切除的质量阈值；
MINLEN，规定read被切除后至少需要保留的长度，如果低于该长度，会被丢掉。

此外，另一个值得注意的地方是，Trimmomatic的报错给出的提示信息都比较难以定位错误问题（如下图），但这往往都只是参数用没设置正确所致。

#### 去除低质量reads

> 进入rawdata所在目录 ，运行程序，示例如下

	[liwen@localhost hd86_nip_hd_171222]$ sh /usr/36T/liwen/shell_script/hiseq_trim.sh rawdata/early_1.fq.gz rawdata/early_2.fq.gz > early.log 2>&1 &

	[liwen@localhost hd86_nip_hd_171222]$ sh /usr/36T/liwen/shell_script/hiseq_trim.sh rawdata/late_1.fq.gz rawdata/late_2.fq.gz > late.log 2>&1 &

### **Annovar软件应用**

#### 安装

下载需edu类邮箱注册，从网上找到一个2018年的下载链接，解压至/opt，即可使用

	http://www.openbioinformatics.org/annovar/download/0wgxR2rIVP/annovar.latest.tar.gz

#### 建立水稻所需库

> 无需root权限建库，这里只是为了便于其他用户今后可以直接使用

	// 创建一个新目录
	[root@localhost annovar]# mkdir ricedb
	// 准备建库所需基因组序列及注释文件（gtf格式）
	##cp /usr/36T/liwen/rice_rawdata/msu/all_genome.gff3 ricedb/msu_genome.gff3
	
	[root@localhost annovar]# cp /usr/36T/liwen/rice_rawdata/rapdb/IRGSP-1.0_genome.fasta ricedb/.
	# 基因组序列信息

#### 1. MSU
	[root@localhost annovar]# awk 'BEGIN{FS=OFS="\t"}{if ($1~/Chr[1-9]$/) {gsub(/Chr/, "chr0", $1); print $0} else if ($1~/1[012]/) {gsub(/C/, "c", $1); print $0}}' /usr/36T/liwen/rice_rawdata/msu/msu_release7.gtf > ricedb/msu_release7.gtf
	# 基因注释信息。修改染色体格式，如Chr1至chr01等
	// gtf转成GenePred文件
	[root@localhost annovar]# gtfToGenePred -genePredExt ricedb/msu_release7.gtf ricedb/Os_refGene_msu.txt
	// 获得各个基因的RNA序列信息
	[root@localhost annovar]# retrieve_seq_from_fasta.pl --format refGene --seqfile ricedb/IRGSP-1.0_genome.fasta ricedb/Os_refGene_msu.txt --out ricedb/Os_refGeneMrna_msu.fa > ricedb/msu_log 2>&1

#### 2. RAPDB
	[root@localhost annovar]# cp /usr/36T/liwen/rice_rawdata/rapdb/IRGSP-1.0_representative_transcript_exon_2018-03-29.gtf ricedb/rapdb_2018.gtf
	# 基因注释信息
	// gtf转成GenePred文件
	[root@localhost annovar]# gtfToGenePred -genePredExt ricedb/rapdb_2018.gtf ricedb/Os-rapdb_refGene.txt
	// 获得各个基因的RNA序列信息
[root@localhost annovar]# retrieve_seq_from_fasta.pl --format refGene --seqfile ricedb/IRGSP-1.0_genome.fasta ricedb/Os-rapdb_refGene.txt --out ricedb/Os-rapdb_refGeneMrna.fa > ricedb/rapdb_log 2>&1

#### 测试

	[liwen@localhost h5-yl_190408]$ awk 'BEGIN{FS=OFS="\t"}NR<=FNR{id=$1"\t"$5; pos[id]=$2; ref[id]=$4; alt[id]=$6}NR>FNR{name=$1"\t"$2; if (name in pos) {$2=pos[name]; $4=ref[name]; $5=alt[name]; print $0}}' output/pool/variant/H5-yl-pool2yl.filter.HC.index_dep10_yl-homo-unique_sig_candidate_rap output/pool/H5-yl-pool2yl.filter.HC.vcf > output/pool/variant/H5-yl-pool2yl.filter.HC.index_dep10_yl-homo-unique_sig_candidate.vcf

	[liwen@localhost h5-yl_190408]$ awk 'BEGIN{FS=OFS="\t"}{print $1,$2,$3,$4,$6}' output/pool/variant/H5-yl-pool2yl.filter.HC.index_dep10_yl-homo-unique_sig_candidate_rap > output/pool/variant/H5-yl-pool2yl.filter.HC.index_dep10_yl-homo-unique_sig_candidate_list

	[liwen@localhost h5-yl_190408]$ head -n 5 output/pool/variant/H5-yl-pool2yl.filter.HC.index_dep10_yl-homo
-unique_sig_candidate_list > tmp_list
	[liwen@localhost h5-yl_190408]$ annotate_variation.pl -out tmp -build Os-rapdb tmp_list /opt/annovar/ricedb > log 2>&1

#### 3. Ensembl
	[liwen@localhost ricedb]$ pwd
	/usr/36T/liwen/rice_rawdata/annovar/ricedb
	// 从Ensembl下载数据
	// 将染色体转成chr格式
	[liwen@localhost ricedb]$ awk 'BEGIN{FS=OFS="\t"}{if ($0~/^#/) {print $0} else if ($1=="1" || $1~/^[2-9]/) {$1="chr0"$1; print $0} else {$1="chr"$1; print $0}}' Oryza_sativa.IRGSP-1.0.43.gtf > Oryza_sativa.ensembl.gtf
	// gtf转成GenePred文件
	[liwen@localhost ricedb]$ gtfToGenePred -genePredExt Oryza_sativa.ensembl.gtf Os-ensembl_refGene.txt
	//基因组序列信息（包括Pt和Mt）
	[liwen@localhost ricedb]$ cp Oryza_sativa.IRGSP-1.0.dna.toplevel.fa Os-ensembl_genome.fa
	// 获得各个基因的RNA序列信息
	[liwen@localhost ricedb]$ retrieve_seq_from_fasta.pl --format refGene --seqfile Os-ensembl_genome.fa Os-ensembl_refGene.txt --out Os-ensembl_refGeneMrna.fa > Os-ensembl.log 2>&1
	// 测试
	[liwen@localhost ricedb]$ cd test
	[liwen@localhost test]$ annotate_variation.pl -out tmp-ensem -build Os-ensembl tmp_list /usr/36T/liwen/rice_rawdata/annovar/ricedb > tmp.log 2>&1