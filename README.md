# How to use iGenSig R modules to build multi-omics models for predicting therapeutic responses based on GDSC dataset

## PLEASE CITE: Wang XS#, Lee S, Zhang H, Tang G, Wang Y. An integral genomic signature approach for tailored cancer therapy using genome-wide sequencing data. Nature Communications. 2022 13(1):2936. https://doi.org/10.1038/s41467-022-30449-7

•	The current version of iGenSig is beta 3.2.9 (Jan 1st, 2022). 

•	The iGenSig was built on R version 4.1.2 and tested on Linux and Windows environment. 

## A. Introduction

•	Here we developed an integral genomic signature (iGenSig) analysis as a new class of transparent, interpretable, and resilient modeling methods for big data-based precision medicine with improved cross-dataset applicability and tolerance to sequencing bias. 

•	We define genomic features that significantly correlate with a clinical phenotype (such as therapeutic response) as genomic correlates, and an integral genomic signature as the integral set of redundant genomic correlates for a given clinical phenotype such as therapeutic response.

•	We postulate that the redundant genomic features, which are usually eliminated through dimensionality reduction or feature removal  during multi-omics modeling, may help overcome the sequencing bias. 

•	Using genomic dataset for chemical perturbations, we developed the iGenSig models predicting cancer cell responses to targeted-therapy agents and tested the cross-dataset performance of selected models based on independent multi-omics datasets for cancer cell lines and cancer patients assessing their therapeutic responses.

## B. How to download R code and data files to run iGenSig

• If you use linux environment, download all files to your local directory. 

```bash
$ git clone https://github.com/wangxlab/iGenSig
```

If you have downloaded all necessary data files, you can find these 4 directories; 

1	“./DrugResponseData” directory

You can find drug response data files (.tsv file) of 2 datasets (GDSC1 and CCLE).

2	“./GenotypeData” directory

You can find genotype data files (.gmt files) of 4 datasets (GDSC,CCLE,BATTLE, and TCGA NSCLC).

3	“./PrecalMatrixData” directory

You can find precalculated feature redundancy data files (.RData files) of 3 datasets (GDSC,CCLE,BATTLE). 

4	“./TestsetAnnotationData” directory

You can find annotation files containing the cell lines names of permutated GDSC test sets for drug ID 1 (.txt file). 


## C. Installing R packages

•	Find "GenSig.modules1.b3.2.9.github.R" file in your folder and see line 2. There is the list of R packages you may have to install unless you did. 

•	If you haven't installed any R pacakges below, it will take about 1 hour. 
```R
packages=c("preprocessCore","pROC","ggplot2","gridExtra","grid","ggpubr","scales","stringr","qvalue","vegan","stringr","tidyverse","pROC","survival","readr","gdata","DescTools","moments","fastcluster","Rfast","dynamicTreeCut","data.table","plyr","dplyr")
sapply(packages,require,character=TRUE)
```

## D. How to build iGenSig models based on GDSC dataset and predict the therapeutic response in CCLE cell lines and patient subjects in the BATTLE trial.

* Find “iGenSig_example_script.R” file in your folder. This file contains the script to perform iGenSig modeling. 

* The iGenSig module has been tested on the latest R version 4.1.2 with Windows 10 computer Intel(R) Core(TM) i7-6700 CPU @ 3.40GHz, 32 GB RAM. 

* Total running time on Windows 10 was ~ 1 hour 20 minutes. It can vary according to your computer's spec.

* Here, we start running iGenSig.
```R
setwd("your working directory")   #set your working directory

source("GenSig.modules1.b3.2.9.github.R")
```

### Step1. Load GDSC drug response data, binary genomic features and feature redundancy files
```R
GDSC.drugData<-read.delim("DrugResponseData/GDSC1_response.tsv",stringsAsFactors = F,check.names = F,header = T,sep="\t")
CCLE.drugData<-read.delim("DrugResponseData/CCLE_response.tsv",stringsAsFactors = F,check.names = F,header = T,sep="\t")
drug.vec=c(1) #select the drug IDs for modeling. drug ID 1 is Erlotinib
load("GenotypeData/GDSC.genotype.list.RData")
load("GenotypeData/CCLE.genotype.list.RData")
GDSC.preCalfile="PrecalMatrixData/GDSC.12bins.preCal.RData"
CCLE.preCalfile="PrecalMatrixData/CCLE.12bins.preCal.RData"
```

### Step2. Specify modeling parameters and create output directory.

```R
#Specify modeling parameters: the q.cut is the cutoff for feature selection, which can be adjusted for different drugs
parameters=list(q.cut=0.1,GDSC.testsetdir="TestsetAnnotationData")
list2env(parameters, .GlobalEnv)
#Create result folders and save parameters
resultdir="Results";dir.create("Results")
GDSC.gensigdir<-paste(resultdir,"/GDSC",sep="")
dir.create(GDSC.gensigdir)
CCLE.gensigdir<-paste(resultdir,"/CCLE",sep="")
dir.create(CCLE.gensigdir)
```

### Step3. Run weighted K-S test to assess the enrichment of genomic features in sensitive or resistant cell lines for a selected drug

```R
test.files<-c()
for (drug.call in drug.vec){
for (i in 1:5) {
outfile=paste(GDSC.gensigdir,"/weightTransf_NES_drugID_",drug.call,"_permuID_",i,".sensitive.xls",sep="")
if(file.exists(outfile)){
next
}
test.files<-append(test.files,paste("testingSet_drugID",drug.call,"_permu",i,".txt",sep=""))
}
}
files.path<-paste(GDSC.testsetdir,test.files, sep="/")
#perform weighted K-S tests for each permuted training/testing set
lapply(files.path,run.weightedKS.fileV2,drugData=GDSC.drugData,genotype.list=GDSC.genotype.list,outdir=GDSC.gensigdir)
#perform weighted K-S tests for all GDSC cell line subjects as training set.
lapply(drugID.call=drug.vec,run.weightedKS.drugV2,drugData=GDSC.drugData,genotype.list=GDSC.genotype.list,outdir=GDSC.gensigdir)
#the weighted KS tests use 2000 permutations to calculate NES scores thus the results could slightly vary between different runs
#the weights calculated in our original study is in the folder: ./Results/GDSC.weights
```

### Step4. Build iGenSig models based on the GDSC dataset.

```R
GDSC.weightdir="./Results/GDSC"
batchCalGenSig.GDSC(GDSC.genotype.list=GDSC.genotype.list,
GDSC.preCalfile=GDSC.preCalfile,
drug.vec=drug.vec,
GDSC.drugData=GDSC.drugData,
GDSC.testsetdir=GDSC.testsetdir,
GDSC.weightdir=GDSC.weightdir,# specify the folder containing the GDSC weight files
GDSC.gensigdir=GDSC.gensigdir,# specify the folder for GDSC output model files
q.cut=q.cut)
```
### Step5. Apply GDSC iGenSig models to model CCLE drug response
```R
batchCalGenSig.validationset (
genotype.list=CCLE.genotype.list,
preCalfile=CCLE.preCalfile,
drug.vec=drug.vec,
GDSC.gensigdir=GDSC.gensigdir,
result.gensigdir=CCLE.gensigdir
)
#benchmark modeling results based on GenSig.sensitive scores
batch.benchmarkGenSig(gensig.dir=CCLE.gensigdir,drugData=CCLE.drugData,dataset="CCLE") # response: the column name of drug response data, which should be either AUC or ActArea
```

### Step6. Apply GDSC model to clinical trial dataset: BATTLE Trial Data as example

```R
Trial.drug=c(1)
load("GenotypeData/TCGA.NSCLC.genotype.list.RData")
load("GenotypeData/BATTLE.genotype.list.RData")
Trial.gensigdir=paste(resultdir,"/BATTLE",sep="");dir.create(Trial.gensigdir)
Trial.preCalfile="PrecalMatrixData/BATTLE.12bins.preCal.RData"
model.GDSC2trial(GDSC.gensigdir=GDSC.gensigdir,
Trial.gensigdir=Trial.gensigdir,
Trial.drug=Trial.drug,
Trial.genotype.list=Trial.genotype.list,
Trial.preCalfile=Trial.preCalfile,
TCGA.catype.genotype.list=TCGA.catype.genotype.list)
## The model.GDSC2trial module can pre-calculate feature redundancy automatically


## Extract binary genomic feature from gene expression data and pre-calculate feature redundancy.
## Here we use BATTLE trial dataset as example. This portion of code should be run before performing iGenSig modeling.
```

### Step 1. Extract binary gene expression features from non-log transformed expression data

```R
##Here we use the gene expression data from BATTLE trial dataset as example

expfile="ExpressionData/BATTLE_GSE33072_ENSG_EXP_unlog.tsv"
expData<-fread(expfile,stringsAsFactors = F, check.names = F,header = T,sep="\t")
expData=data.frame(row.names = expData$Name,expData[,-1],stringsAsFactors = F, check.names = F)
outfilepath<-paste0(sub("(.*?).tsv", "\\1", expfile),".12levelFeatures.gmt",sep="")
binarize.expfeature(expData,outfilepath,sdpercfilter=0.2,sdlevels=-6:6,genomewide=TRUE)
```

### Step2. Extract pre-calculate feature redundancy for GDSC genomic features based on their co-ocurrance in the Pan-cancer TCGA dataset. 
```R
Trial.genotype.list=read_gmt(outfilepath,min=1)
load("GenotypeData/TCGA.NSCLC.genotype.list.RData")
Trial.preCalfile=paste0(sub(".gmt$","",outfilepath),".preCal.RData")
batch_calSimilarity_genolist(subject.genotype.list=Trial.genotype.list,REF.genotype.list=TCGA.catype.genotype.list,subject.preCalfile=Trial.preCalfile)
```



## E. Expected outcome 

* Please find "./Results/CCLE/benchmark.genSig.result.xls" file. The file has the summary of the tested drug name and prediction performance AUROC. 

| drugID | drugName  | permuID | AUC        |
| ------ | --------- | ------- | ---------- |
| 1      | Erlotinib | 1       | 0.81589806 |
| 1      | Erlotinib | 2       | 0.81185275 |
| 1      | Erlotinib | 3       | 0.81565534 |
| 1      | Erlotinib | 4       | 0.8086165  |
| 1      | Erlotinib | 5       | 0.81229773 |
