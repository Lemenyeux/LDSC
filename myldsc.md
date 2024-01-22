# Download LDSC data
```
cd /mnt/ldsc/myldsc

## GWAS summary data
wget -c http://www.broadinstitute.org/collaboration/giant/index.php/GIANT_consortium_data_files
gunzip -k GIANT_BMI_Speliotes2010_publicrelease_HapMapCeuFreq.txt.gz
## 基线模型 baselineLD model LD scores
tar -xvzf LDSCORE_1000G_Phase1_baseline_ldscores.tar
## 权重数据
tar -xvzf LDSCORE_weights_hm3_no_hla.tar
## 位点频率数据
tar -xvzf LDSCORE_1000G_Phase1_frq.tar
## 千人参考基因组基因型数据
tar -xvzf LDSCORE_1000G_Phase3_plinkfiles.tar
## HapMap3 SNP的连接列表
bzip2 -d LDSCORE_w_hm3.snplist.bz2
### 输出hapmap3 snp rsid列表
awk '{if ($1!="SNP") {print $1} }' LDSCORE_w_hm3.snplist > listHM3.txt
## 千人基因组细胞类型数据
tar -xvzf LDSCORE_1000G_Phase1_cell_type_groups.tar
```
# 查看文件内容
## GWAS summary data：rs号，allele1，allele2，位点频率，P值和样本量N

![image](https://github.com/Lemenyeux/LDSC/assets/87812974/ebf7b738-4541-419e-a041-929056d27925)
## LDSC基线模型
分染色体，每个染色体有三个文件：注释文件`baseline.1.annot.gz`, LD score文件`baseline.1.l2.ldscore.gz`, M_5_50文件`baseline.1.l2.M_5_50`
##### 注释文件`baseline.1.annot.gz`
具有功能注释分类如`Coding_UCSC`，每个SNP位点是否属于该分类

![image](https://github.com/Lemenyeux/LDSC/assets/87812974/100b634f-d9b5-4731-8b77-06fde6156450)
##### LD score文件`baseline.1.l2.ldscore.gz`
各个功能注释下每个SNP的LD score

![image](https://github.com/Lemenyeux/LDSC/assets/87812974/fa775c9c-04f1-4d49-af5f-5d2bda110236)
#####  M_5_50文件`baseline.1.l2.M_5_50`
一行，列数和注释文件功能分类数目相同，每列为对应注释类别中MAF>5%的snp个数（比如染色体1注释文件有711594个SNP，其中Coding_UCSC分类下有450756个SNP的MAF>5%）

![image](https://github.com/Lemenyeux/LDSC/assets/87812974/792e6625-ef88-4d5f-9895-3dccdc53caee)
## 权重数据
每个染色体一个LD score文件作为权重，统计方面的考虑，如`weights.1.l2.ldscore.gz`, 文件包含染色体编号、SNP号、物理位置BP、遗传坐标CM、MAF和LD score

![image](https://github.com/Lemenyeux/LDSC/assets/87812974/e65bcd6d-6ef3-4e79-931f-6ef13095336b)
## 千人基因组位点频率数据
每个染色体一个文件，如`1000G.mac5eur.1.frq.gz`，包含染色体编号、SNP号、等位基因A1和A2，等位基因A1频率；该文件用于后期筛选MAF>5%的SNP

![image](https://github.com/Lemenyeux/LDSC/assets/87812974/ba0291f4-503b-42ff-8c0f-0c8cce74a53a)
## 千人参考基因组基因型数据
每个染色体的bed, fam和bim文件
## HapMap3 SNP的连接列表 LDSCORE_w_hm3.snplist
HapMap3的SNP的rs号，A1和A2信息，用于提取只包含在HapMap3中的SNP位点（1217312个SNP位点）

![image](https://github.com/Lemenyeux/LDSC/assets/87812974/6fc72cd2-feb5-4ec7-af16-8ee1817d40f8)

# 生成sumstat文件
将GWAS summary数据和LDSCORE_w_hm3.snplist文件进行munge，输出格式化后的sumstat文件`BMI.sumstats.gz`
在我的服务器上计算过程好久，因此后面的计算基于已有的sumstat数据`UKB_20022_Birth_weight.sumstats`进行分析
```
/mnt/ldsc/munge_sumstats.py --sumstats GIANT_BMI_Speliotes2010_publicrelease_HapMapCeuFreq.txt\
--merge-alleles LDSCORE_w_hm3.snplist\
--out BMI\
--a1-inc
## --a1-inc: A1 is the increasing allele.
```
`UKB_20022_Birth_weight.sumstats`数据情况，包含SNP号，A1，A2，样本量N和summary统计量Z-score

![image](https://github.com/Lemenyeux/LDSC/assets/87812974/c2fcf337-69c7-4c7b-9768-c71fbbe687e1)

# Partition heritability
```
/mnt/ldsc/ldsc.py --h2 UKB_20022_Birth_weight.sumstats/
--ref-ld-chr baseline/baseline./
--w-ld-chr weights_hm3_no_hla/weights./
--overlap-annot/
--frqfile-chr 1000G_frq/1000G.mac5eur./
--out Bweight_baseline
## --overlap-annot: read in the .annot.gz files.
```
# 结果解释
 - Lambda GC is median(chi^2)/0.4549.
 - Mean chi^2 is the mean chi-square statistic.
 - Intercept is the LD Score regression intercept. The intercept should be close to 1, unless the data have been GC corrected, in which case it will often be lower.
 - Ratio is (intercept-1)/(mean(chi^2)-1), which measures the proportion of the inflation in the mean chi^2 that the LD Score regression intercept ascribes to causes other than polygenic heritability. The value of ratio should be close to zero, though in practice values of 10-20% are not uncommon, probably due to sample/reference LD Score mismatch or model misspecification (e.g., low LD variants have slightly higher h^2 per SNP)

![image](https://github.com/Lemenyeux/LDSC/assets/87812974/a1e95928-f376-45c6-9861-a92f4f477192)

结果在`.results file`文件中，注释类别（比如DHS,Coding等）、SNPs 占比、遗传度占比、Enrichment、Enrichment标准误、Enrichment P值、回归系数、回归系数标准误、回归系数Z值

![image](https://github.com/Lemenyeux/LDSC/assets/87812974/95c255d3-dad5-476b-997b-4ec28ec7ec2c)
