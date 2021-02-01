  
    2  conda activate qiime2-2020.11
    4  mkdir atacama
    5  cd atacama

    #由于序列数据比较大，为了缩短练习时间，此处不上传学习用的数据
    #先下载需要使用的数据，具体下载链接见https://docs.qiime2.org/2020.11/tutorials/atacama-soils/
    #注释文件gg-13-8-99-515-806-nb-classifier.qza 在moving-picture教程中有

  
    7  mkdir -p emp-paired-end-sequences
    8  #双端数据导入，数据建库类型为EMP双端序列
    9  qiime tools import --type EMPPairedEndSequences --input-path emp-paired-end-sequences --output-path emp-paired-end-sequences.qza
   10  #根据Barcode序列信息进行样品拆分。此时需要样本元数据文件sample-metadata.tsv,必须指明该文件的哪一列包含每个样本的条形码
   11  #序列取反向互补(barcode加有右端时使用)
   13  mkdir 1_demuxed
   12  qiime demux emp-paired --m-barcodes-file sample-metadata.tsv --m-barcodes-column barcode-sequence --p-rev-comp-mapping-barcodes --i-seqs emp-paired-end-sequences.qza --o-per-sample-sequences 1_demuxed/demux-full.qza --o-error-correction-details 1_demuxed/demux-details.qza
   14  #结果可视化
   15  qiime demux summarize --i-data 1_demuxed/demux-full.qza --o-visualization 1_demuxed/demux-full.qzv
   
   
   16  #对数据重采样(非必要步骤)。此处是为了加快教程运行时间，另一个是为了演示功能。因此，如果没有充分的理由，不需要重采样。
   17  qiime demux subsample-paired --i-sequences 1_demuxed/demux-full.qza --p-fraction 0.3 --o-subsampled-sequences 1_demuxed/demux-subsample.qza
   18  #对重抽样的结果可视化
   19  qiime demux filter-samples --i-demux 1_demuxed/demux-subsample.qza a
   20  qiime demux summarize  --i-data 1_demuxed/demux-subsample.qza --o-visualization 1_demuxed/demux-subsample.qzv
   21  #观察到抽样统计表中最后有些样本的序列数少于100，因此将这一部分数据过滤掉
   22  qiime tools export --input-path 1_demuxed/demux-subsample.qzv --output-path 1_demuxed/demux-subsample/
   24  qiime demux filter-samples --i-demux 1_demuxed/demux-subsample.qza --m-metadata-file 1_demuxed/demux-subsample/per-sample-fastq-counts.tsv --p-where 'CAST([forward sequence count] AS INT)>100' --o-filtered-demux 1_demuxed/demux.qza
  
   26  qiime demux summarize  --i-data 1_demuxed/demux.qza --o-visualization 1_demuxed/demux.qzv
   27  #通过查看后，决定从13bp处开始修建，然后右端选择150，使正反向数据具有较好的重合
   
   25  #去噪并生成特征表和代表序列
   30  mkdir 2_QC   
   29  qiime dada2 denoise-paired --i-demultiplexed-seqs 1_demuxed/demux.qza --p-trim-left-f 13 --p-trim-left-r 13 --p-trunc-len-f 150 --p-trunc-len-r 150 --o-table 2_QC/table.qza --o-representative-sequences 2_QC/rep-seqs.qza --o-denoising-stats 2_QC/denoising-stats.qza
   31  #生成结果摘要
   32  qiime feature-table summarize --i-table 2_QC/table.qza --o-visualization 2_QC/table.qzv --m-sample-metadata-file sample-metadata.tsv 
   33  qiime feature-table tabulate-seqs --i-data 2_QC/rep-seqs.qza --o-visualization 2_QC/rep-seqs.qzv
   34  qiime metadata tabulate --m-input-file 2_QC/denoising-stats.qza --o-visualization 2_QC/denoising-stats.qzv

   47  #通过查看table.qzv文件，可以看到有很多样本测序量在1000以下。一般可以选择大值，尽可能扔掉样本，从而保留质量高的结果发现规律。如果不够，再降低这个值，保留更多的样本。在此案例中，我们选择保留1000以上的样本，因此选择989这个数字。
   48  #进化树构建和多样性分析
   49  mkdir 3_phylogenetic
   51  time qiime phylogeny align-to-tree-mafft-fasttree --i-sequences 2_QC/rep-seqs.qza --o-alignment 3_phylogenetic/aligned-rep-seqs.qza --o-masked-alignment 3_phylogenetic/masked-aligned-rep-seqs.qza --o-tree 3_phylogenetic/unrooted-tree.qza --o-rooted-tree 3_phylogenetic/rooted-tree.qza
   
   52  #重采样，采样Q=989
   53  qiime diversity core-metrics-phylogenetic --i-phylogeny 3_phylogenetic/rooted-tree.qza --i-table 2_QC/table.qza --p-sampling-depth 989 --m-metadata-file sample-metadata.tsv --output-dir 4_core-metrics-results

   54  #尝试几种可以统计连续型变量的统计方法
   55  #qiime diversity mantel可以基于特征的距离矩阵，和样本元数据的距离矩阵，计算两者间的相关性。
   56  #由于在上面的操作中，我们进行了重取样，因此可能会出现实验设计中存在缺失样本的情况。所以此处先生成一个完整信息的海拔列
   57  qiime metadata distance-matrix --m-metadata-file sample-metadata.tsv --m-metadata-column elevation --o-distance-matrix 5_diversity/sample-metadata-elevation.qza
   58  #环境因子关联分析
   60  qiime diversity mantel --i-dm1 4_core-metrics-results/weighted_unifrac_distance_matrix.qza --i-dm2 5_diversity/sample-metadata-elevation.qza --p-method spearman --p-intersect-ids True --p-label1 weighted_unifrac --p-label2 elevation --o-visualization 4_core-metrics-results/weighted_unifrac_evaluation.qzv
   61  #qiime diversity bioenv 计算元数据的欧式距离中哪一类与距离矩阵秩最大相关。其中所有的数字列都会考虑，缺失值会被自动移除。原始数据中列太多，也存在错误，只提取其中部分
   62  cut -f 1-5,10,13 sample-metadata.tsv > temp
   63  #计算数字列的相关性
   65  qiime diversity bioenv --i-distance-matrix 4_core-metrics-results/weighted_unifrac_distance_matrix.qza --m-metadata-file temp --o-visualization 4_core-metrics-results/weighted_unifrac_bioenv.qzv
   66  #可以看到结果中average-soil-relative-humidity与很多因素相关，是最大相关因素

   67  #aplha多样性与连续性变量分析
   68  #以样本丰富度为例(observed_otus(richness))
   69  qiime diversity alpha-correlation --i-alpha-diversity 4_core-metrics-results/observed_features_vector.qza --m-metadata-file sample-metadata.tsv --o-visualization 4_core-metrics-results/observed_features_correlation.qzv

   70  #物种组成差异及相关分析
   71  #门水平下查看不同土壤相对温度下微生物组成，哪个门最高？哪些种类与湿度正/负相关？
   72  #先进行物种注释，再统计分析
   
   74  mkdir 5_Taxonomy  
   73  qiime feature-classifier classify-sklearn --i-classifier gg-13-8-99-515-806-nb-classifier.qza --i-reads 2_QC/rep-seqs.qza --o-classification 5_Taxonomy/taxonomy.qza
   75  #生成物种可视化表
   76  qiime metadata tabulate --m-input-file 5_Taxonomy/taxonomy.qza --o-visualization 5_Taxonomy/taxonomy.qzv
   77  #物种组成柱状图，按level2和sample-type排序
   78  qiime taxa barplot --i-table 2_QC/table.qza --i-taxonomy 5_Taxonomy/taxonomy.qza --m-metadata-file sample-metadata.tsv --o-visualization 5_Taxonomy/taxa-bar-plots.qzv
   
   79  #在有无植被的取样地点，什么菌门差异明显？
   80  #按门比较，需要先合并
   81  qiime taxa collapse --i-table 2_QC/table.qza --i-taxonomy 5_Taxonomy/taxonomy.qza --p-level 2 --o-collapsed-table 5_Taxonomy/table-l2.qza
   82  #格式转换
   83  qiime composition add-pseudocount --i-table 5_Taxonomy/table-l2.qza --o-composition-table 5_Taxonomy/comp-table-l2.qza
   84  #按有无植被差异比较
   85  qiime composition ancom --i-table 5_Taxonomy/comp-table-l2.qza --m-metadata-file sample-metadata.tsv --m-metadata-column vegetation --o-visualization 5_Taxonomy/l2-ancom-vegetation.qzv
