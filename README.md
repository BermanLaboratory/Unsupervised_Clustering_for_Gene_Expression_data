# Gene-Expression-Unsupervised-Clusteing
Using consensus clustering approach to find pattern in expression data sets

Unsupervised class discovery is a data mining method to identify unknown possible groups (clusters) of items solely based on intrinsic features and no external variables. Basically clustering includes 4 steps:

#### [1] Data preparation,
#### [2] Dissimilarity matrix calculation,
#### [3] applying clustering algorithms, 
#### [4] Assessing cluster assignment
_________________________________________________________________________________________________________________________________________________________________________________________

### [1] Data preparation

In this step we need to filter out incomplete cases and low expressed genes, then transforming/normalizing gene expression values. Usually analysis would start from a raw count matrix comming from an RNA-seq experiment. Because there are a large number of features (gene) in such matrix , a feature selection step should be done to limit the analysis to those genes that possibly explain variation between samples in the cohort.  
To do so ;
- For filteration: I keep those genes  that  have expression in 10% of samples with a count of 10 or higher. 
- For transformation/normalization : I  use variance stabilizing transformation (VST). I use ```vst``` function from ```DESeq2 packages``` which at the same time will normalize the raw count also. Using other type of transformation like Z score, log transformation are also quiet commmon.
- For feature selection: I  select 2k, 4k and 6k top genes based on median absolute deviation (MAD) . A number of other methods like "feature selection based on the most variance", "feature dimension reduction and extraction based on Principal Component Analysis (PCA)", and "feature selection based on Cox regression model" could be applied. See bioconductor package [CancerSubtypes manual](http://www.bioconductor.org/packages/release/bioc/html/CancerSubtypes.html). 

Other approaches also could be use for example: 
 - Using log2(RSEM) gene expression value and removing genes with NA values more than 10% across samples. Then selected top 25% most-varying genes by standard deviation of gene expression across samples ([A. Gordon Robertson et al., Cell,2017](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5687509/)). 

 - Using FPKM matrix and keeping genes with (log2(FPKM+1)>2 in at least 10% of samples and selecting a subsets (2K, 4K, 6K ) of MAD ranked genes to identify stable classes ([Jakob Hedegaard et al, Cancer Cell, 2016](https://www.sciencedirect.com/science/article/pii/S1535610816302094#mmc1)). 
 

```R
#_________________________________reading data and processing_________________________________#
# reading count data
rna <- read.table("Uromol1_CountData.v1.csv", header = T, sep = ",")
head(rna[1:5, 1:5], 5)
#                   U0001 U0002 U0006 U0007 U0010
#ENSG00000000003.13  1458   228  1800  3945   293
#ENSG00000000005.5      0     0     9     0     0
#ENSG00000000419.11   594    23   792  1378   139
#ENSG00000000457.12   548    22  1029   976   148
#ENSG00000000460.15    53     2   190   136    47

dim(rna)
# [1] 60483   476  this is a typical output from hts-seq count matrix with more than 60,000 genes

# dissecting dataset based on gene-type
library(biomaRt)
mart <- useDataset("hsapiens_gene_ensembl", useMart("ensembl"))
genes <- getBM(attributes= c("ensembl_gene_id","hgnc_symbol", "gene_biotype"), 
               mart= mart)

# see gene types returned by biomaRt
data.frame(table(genes$gene_biotype))[order(- data.frame(table(genes$gene_biotype))[,2]), ]

# we will continue with protein coding here:
rna <- rna[substr(rownames(rna),1,15) %in% genes$ensembl_gene_id[genes$gene_biotype == "protein_coding"],]

dim(rna)
#[1] 19581   476 These are protein coding genes

# reading associated clinical data
clinical.exp <- read.table("uromol_clinic.csv", sep = ",", header = T)
head(clinical.exp[1:5,1:5], 5)
#      UniqueID   CLASS  BASE47    CIS X12.gene.signature
#U0603    U0603 luminal luminal no-CIS          high_risk
#U0497    U0497 luminal   basal no-CIS           low_risk
#U0839    U0839 luminal luminal    CIS           low_risk
#U1043    U1043 luminal luminal no-CIS          high_risk
#U0566    U0566 luminal   basal no-CIS           low_risk

#______________________ Data tranformation & Normalization ______________________________#
library(DESeq2)
dds <- DESeqDataSetFromMatrix(countData = rna,
                              colData = clinical.exp,
                              design = ~ 1) # 1 passed to the function because of no model
# Prefilteration: Not necessary
# Remove those have lower than 10 count across all samples
keep <- rowSums(counts(dds)) >= 10
dds <- dds[keep,]

# OR:
# at 10% of samples with a count of 10 or higher
keep <- rowSums(counts(dds) >= 10) >= round(ncol(rna)*0.1)
dds <- dds[keep,]


# vst tranformation
vsd <- vst(dds) # For a fully unsupervised transformation, 
                # one can set blind = TRUE (which is the default).

#head(assay(vsd), 3)
#colData(vsd)






