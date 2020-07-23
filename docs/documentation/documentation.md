__主要的三部分模块都是可以独立运行的，只要提供相应的输入数据__

### 1. RNA-seq数据处理

####  使用
```
mkdir s1_pipeone_raw
cd s1_pipeone_raw
bash /your/path/to/PipeOne/s1_pipeOne.sh --reads "../test_dat/s1_RNA-seq/*.R{1,2}.fastp.fq.gz" --genome hg38 --cleaned true
```
#### 参数
Require:
```
--reads  <string>     输入FASTQ文件, for example: "/home/reads/*_R{1,2}.fastq.gz"，
                      * 代表可以匹配的文件， {1,2} 代表双端测序

--genome <string>     使用的在 conf/igenomes.config 定义genome 版本
```
Optional:
```
--cleaned  <booloen>    true 或者 false FASTQ文件是否是质量控制过，如果是false 则用fastp进行质量控制。defalt [false]
--layout   <string>     paired 或者 single. 默认 [paired]
--threads  <int>       每一步使用的最大线程数。 默认 [8]
--maxForks <int>       并行多少步骤。 默认 [2]
--saveIntermediateFiles       是否保留中间文件到结果。 默认 [off]
--saveIntermediateHisat2Bam   是否保留hisat2 的bam 文件. 默认 [off]
-h --help     显示帮助
```


#### 输出

* __RNA-seq 经过不同的程序运行的结果__
    * 00_tables 经过不同的程序运行得到的表格
    * s1.*文件夹, 经过不同的程序运行目录
      * result 结果目录
      * work 工作目录
```
$ tree -L 2
.
├── 00_tables
│   ├── 00_rawdata
│   ├── s1.1_lncR_mRNA
│   ├── s1.3_APA-3TUR
│   ├── s1.4_retrotranscriptome
│   ├── s1.5_fusion
│   ├── s1.6_rnaEditing
│   ├── s1.7_alternative_splicing
│   └── s1.8_SNP
├── s1.1_lncRNA
│   ├── results
│   └── work
├── s1.3_APA-3TUR
│   ├── results
│   └── work
├── s1.4_retrotranscriptome
│   ├── results
│   └── work
├── s1.5_fusion
│   ├── results
│   └── work
├── s1.6_rnaEditing
│   ├── results
│   └── work
├── s1.7_alternative_splicing
│   ├── results
│   └── work
└── s1.8_SNP
    ├── results
    └── work

30 directories, 0 files
```

* __RNA-seq 经过不同的程序运行得到的表格: s1.*文件夹__
```
$ ls  00_tables
00_rawdata
s1.1_lncR_mRNA
s1.3_APA-3TUR
s1.4_retrotranscriptome
s1.5_fusion
s1.6_rnaEditing
s1.7_alternative_splicing
s1.8_SNP
```


* __RNA-seq 数据处理后得到的各个表格汇总，为下一步运行所需__
```
$ ls 00_tables/00_rawdata
APA_pau-distal-proximal.csv
lncR_gene.tpm.csv
merge_graphs_alt_3prime_C3.confirmed.psi.csv
merge_graphs_alt_5prime_C3.confirmed.psi.csv
merge_graphs_exon_skip_C3.confirmed.psi.csv
merge_graphs_intron_retention_C3.confirmed.psi.csv
merge_graphs_mult_exon_skip_C3.confirmed.psi.csv
merge_graphs_mutex_exons_C3.confirmed.psi.csv
prot_gene.tpm.csv
retro-FPKM-divide_totalMapReads.csv
RNA-editing-rate.csv
```



### 2. 寻找重要特征

#### 需要
1. sample_info
sample_info 必须有两列信息, 列名为 Sample 和Group
例如
```
$ cat sample_info.csv
Sample,Group
sample_1,0
sample_2,0
sample_3,1
sample_4,1
sample_n,0
```
注意：group的值只有两组

2. 包含个类型数据的文件夹

```
$ cat table.csv
feature_id,sample_1,sample_2,sample3...,sample_n
feature_1,value,value,value,value,...,value
feature_2,value,value,value,value,...,value
feature_3,value,value,value,value,...,value
feature_n,value,value,value,value,...,value
```

注意每个文件都是csv逗号分割格式。

#### 一步运行
```
mkdir s2_randomForest
cd s2_randomForest
nextflow run /your/path/to/PipeOne/s2_ml_randomForest.nf  -profile docker --sample_info ../test_dat/s2_tables/s1_sample_info-tumor-normal.csv --tables ../test_dat/s2_tables/00_rawdata --var_topK 1000  --threads 8
```



#### 分步骤运行
* 取 top N variance features
  
```
source activate pipeOne_ml
baseDir="/your/path/to/PipeOne/"
python3 ${baseDir}/bin/ML/proc_raw_data.py proc --rawdir 00_rawdata  --sample_info s1_sample_info-tumor-normal.csv --var_topk 1000 --tdir ./data/proc/
    
```
--rawdir RNA-seq各个结果的表格，例如表达水平（TPM值），RNA-editing rate, Fusion event 等
--sample_info 样品信息
--var_topk 各表格用多少feature 数，默认1000
--tdir 输出目录， 默认 ./data/proc/

* 讲数据分为测试集和训练集
```
python3 ${baseDir}/bin/ML/proc_raw_data.py train_test_split --indir ../data/proc  --sample_info s1_sample_info-tumor-normal.csv
```
--indir 上一步的输出结果
--sample_info 样品信息


* 运行主程序
```
python3 ${baseDir}/bin/ML/main_randomForest.py --threads 8
```
--threads 用多少线程


* 整理结果（可选）
```
python3 ${baseDir}/bin/ML/result_summary.py feature  --rf_res_fi data/feature_importance.csv  --ginfo_fi ../../s1.1_lncRNA/results/novel_lncRNA/protein_coding_and_all_lncRNA.info.tsv
```
--rf_res_fi  Random Foreast 的结果（data/feature_importance.csv）
--ginfo_fi 基因的信息文件，可在第一步的结果里面找到（s1_lncRNA/results/novel_lncRNA/protein_coding_and_all_lncRNA.info.tsv）

#### __结果文件__:

* results/data/feature_importance.csv

* results/data/feature_importance-addName.csv

* results/data/discriminative_power_of_topk_feature.csv


### 3. 亚型分析