# ChloroDBPFinder
[![Generic badge](https://img.shields.io/badge/ChloroDBPFinder-ver_1.0-<COLOR>.svg)](https://github.com/TingtingZhao81/ChloroDBPFinder)
![Maintainer](https://img.shields.io/badge/maintainer-Tingting_Zhao-blue)

   
`ChloroDBPFinder` is an R package to identify Cl-containing compounds in untargeted LC-HRMS analysis. 
It incorporated a suite of state-of-art features, such as machine learning, false positive detection and molecular networking
to facilitate unknown annotation.

The package is written in the language R and its source code is publicly available at [ChloroDBPFinder](https://github.com/TingtingZhao81/ChloroDBPFinder).

<!-- TOC -->
* [ChloroDBPFinder](#chlorodbpfinder)
  * [Installation instructions](#installation-instructions)
    * [System requirements](#system-requirements-)
    * [ChloroDBPFinder installation](#chlorodbpfinder-installation)
    * [Download machine learning model](#download-machine-learning-model)
  * [Instructions for package usage](#instructions-for-package-usage)
    * [Data preparation preparation](#data-preparation-preparation)
    * [Set parameters](#set-parameters)
    * [Part 1 Extraciton of chlorinated compounds](#part-1-extraciton-of-chlorinated-compounds)
    * [Part 2 Alignment across samples](#part-2-alignment-across-samples)
    * [Part 3 Missing value imputation](#part-3-missing-value-imputation-)
    * [Part 4 Annotation](#part-4-annotation)
  * [Citation](#citation)
<!-- TOC -->


## Installation instructions
### System requirements 
R version 4.2.0 or above is required. To install ChloroDBPFinder package successfully, please install the following packages first:

```angular2html
install.packages('devtools')
install.packages('doParallel')
install.packages('foreach')
install.packages('parallel')
install.packages('clue')
install.packages('randomForest')
install.packages("BiocManager")
BiocManager::install("xcms")

```
In order to install ISFrag successfully to identify in-source fragments, please install the following packages first:
```angular2html
BiocManager::install("CAMERA")
BiocManager::install("metaMS")
devtools::install_github('HuanLab/ISFrag')
```
If you can not install ISFrag successfully, 
you can simply bypass ISFrag by setting the parameters `isfrag = FALSE` in line 22 of demo script.


### ChloroDBPFinder installation
```angular2html
devtools::install_github('TingtingZhao81/ChloroDBPFinder')
```

### Download machine learning model
`Binary classifier` and `Multiclass classifier` can be freely downloaded in [machine learning model website](https://github.com/TingtingZhao81/ChloroDBPFinder_support/tree/main/machine_learning_model)

`Demo data` can be freely downloaded in [demo data website](https://github.com/TingtingZhao81/ChloroDBPFinder_support/tree/main/demo_data)

## Instructions for package usage

ChloroDBPFinder contains four modules: <br>
[1. Extration of chlorinated compounds](#Part-1-Identification-of-chlorinated-compounds) <br>
[2. Alignment across samples](#Part-2-Alignment-across-samples) <br>
[3. Missing value imputation](#Part-3-Evidence-based-missing-value-imputation) <br>
[4. Annotation](#Part-4-Annotation) <br>

### Data preparation preparation
1. Put mzML or mzXML format raw lcms data in a folder
2. If users want to use customized feature table, please prepare the feature table in advance. <br>
   The format for customized feature table:<br>
- 'featureID': ID of the features
- 'mz': m/z of the features
- 'rt': retention time in seconds
- 'Intensity': peak intensity, peak height(prefered)/area
- 'sample': sample ID, order of the corresponding raw lcms file
- If user want to identify in source fragment, 'mzmin', 'mzmax', 'rtmin', 'rtmax' are required. <br>
- If users want to conduct compound annotation, 'ms2_mz' and 'ms2_int' are required.<br>


| featureID | mz       | mzmin    | mzmax    | rt      | rtmin   | rtmax   | into   | intb   | Intensity | sn  | sample | ms2_ mz                  | ms2_int             |
|-----------|----------|----------|----------|---------|---------|---------|--------|--------|-----------|-----|--------|--------------------------|---------------------|
| 1         | 327.0745 | 327.0743 | 327.0750 | 1813.34 | 1789.08 | 1838.88 | 100000 | 100000 | 100000    | 200 | 1      | 70.0292;88.0397;111.0548 | 3140;10889;855;2921 | 
| 2         | 274.2744 | 274.2741 | 274.2747 | 1821.80 | 1810.46 | 1850.32 | 20000  | 200000 | 200000    | 100 | 1      | 118.0656;219.0736        | 3979;465            |


3. If users want to construct a molecular networking, please prepare a reference(seed) table containing known compounds or precursors <br>
The reference table should be in a format as below. Users can call `data('seed_demo')` to check the format. <br>
- 'featureID': match the ID in the Cl-containing features table, if this compound exist in the feature table
- 'label': seed
- 'annotation': the name of this compound
- 'MF': molecular formula of this compound
- 'mz': m/z 
- 'rt': retention time in seconds, optional
- 'cl': number of Cl elements in this compound
- 'ms2_mz': m/z of fragment ion in MS/MS
- 'ms2_int': intensity of fragment ion in MS/MS.

| featureID  | label | annotation     | MF           | mz       | rt   | cl | ms2_mz                   | ms2_int             |
|------------|-------|----------------|--------------|----------|------|----|--------------------------|---------------------|
| Precursor1 | seed  | APM            | C14H18N2O5   | 295.1296 | 520  | 0  | 70.0292;88.0397;111.0548 | 3140;10889;855;2921 |
| S1_2       | seed  | chlorinatedAPM | C14H17N2O5Cl | 329.0906 | 1200 | 1  | 7002;119.0735;120.08     | 789;9909;1230       |

4. If users want to construct a molecular networking, users can either use a [default reaction pathway table](https://github.com/TingtingZhao81/ChloroDBPFinder_support/tree/main/reaction_pathways) or customize one. <br>
Users can call `data('reaction_pathways')` to check the format. <br>

### Set parameters
A demo script can be downloaded from [demo script website](https://github.com/TingtingZhao81/ChloroDBPFinder_support/tree/main/demo_script)

* Load library 
  ```angular2html
  library(ChloroDBPFinder)

* Specify the path of machine learning model
  ```angular2html
  # Users need to change the path of multiclass classifier
  binary_model_file <- "C:/Users/User/ChloroDBPFinder/binary_model.rds"
  multi_model_file <- "C:/Users/User/ChloroDBPFinder/multiclass_model.rds"
  ```
* Specify the path of raw lcms data
  ```angular2html
  mzmldir <- "C:/Users/User/Desktop/package_devolopment_notes"
  ```
* Specify the format of the raw lcms data. 
  ```angular2html
    lcmspattern <- ".mzXML"  # String: ".mzML" or ".mzXML"
    mzMLfile <- list.files(mzmldir, pattern = lcmspattern)
    ```

* Specify whether use a feature table generated from other software. 
    ```angular2html
    use_customized_table <- FALSE
    # use_customized_table = TRUE, change the path and name of the customized table
    customized_table <- 'C:/Users/User/Desktop/my_customized_feature_table.csv'
    ```
    
* Specify whether detect in-source fragment or not.
    ```angular2html
  isfrag <- TRUE # Boolean: TRUE or FALSE
    ```
* Specify the path of the MS/MS spectra database, if users want to conduct database searching for compound annotation
   ```
  Cl_db_path <- "C:/Users/User/Tingting/2022-11-03-Cl_project/ChloroDBP Hunter/06-02/Cl_compounds_in_NIST.csv"
  ```
* Specify path of the reference table containing the known compounds or precursors, if users want to construct a molecular networking
  ```
  ref_path <- "C:/Users/User/Desktop/reference_table.csv"
  ```


### Part 1 Extraciton of chlorinated compounds


* load customized feature table or extract all chemical features

  ```angular2html
  if(use_customized_table){
    peaks <- read.csv(customized_table)
  }else{
   peaks <- extractPeak(mzMLdirectory = mzmldir, mzMLfile = mzMLfile[1], SN= 20, noise =2000, rt_min =300, rt_max =3000 )
   write.csv(peaks, paste0(mzmldir,"/",strsplit(mzMLfile[1], split = lcmspattern)[[1]][1], "_",nrow(peaks), "_peaks_with_MS2.csv"),row.names = FALSE )
  }
    ```
  - SN: signal to noise ratio, user can decrease this value to improve sensitivity. Default: 20
  - noise: intensity threshold, peaks with intensity lower than this threshold will be removed, default: 2000
  - rt_min: retention time in seconds, peaks with rt lower than this threshold will be removed, default: 300
  - rt_max: retention time in seconds, peaks with rt higher than this threshold will be removed, default: 3000
  
* determine chlorinated compounds
  ````
  binary_rf_model <- readRDS(binary_model_file) # load binary classifier
  multi_rf_model <- readRDS(multi_model_file)   # load multiclass classifier
  xcmsrawlcms <- eicRawlcms(mzMLdirectory = mzmldir, mzMLfile = mzMLfile[i]) # pre-process the raw lcms for EIC extracion
  cl_tb <- selectCl(mzMLdirectory = mzmldir, mzMLfile = mzMLfile[1], original_ft = peaks,
                     binary_model = binary_rf_model, multi_model = multi_rf_model, ms1_spectra_rt_tol =20, ms1_spectra_mz_tol = 25, 
                     iso_mass_diff_1 = 1.003355, iso_mass_diff_2 = 1.99705, iso_mass_diff_3 = 3, iso_mass_diff_4 = 3.994)
  ````
  
* identify in source fragment based on ISFrag package, details about ISFrag usage can be found in [ISFrag](https://github.com/HuanLab/ISFrag)
  ````
  # detect in-source fragment 
  if(isfrag){ 
     library(ISFrag)
     customFT <- cl_tb
     customFT$Adduct <- 0
     customFT$isotope <- 0
     rownames(customFT) <- peaks$featureID
     if(grepl("mzXML", mzMLfile[1])){filename <- strsplit( mzMLfile[1], split=".mzXML")[[1]][1]}
  else{filename <- strsplit( mzMLfile[1], split=".mzML")[[1]][1]}
     ISFdirectory_name <- paste0(mzmldir,"/inSourceFrag_", filename)
     dir.create(ISFdirectory_name)
     file.copy(from = paste0(mzmldir, "/", mzMLfile[1]), to = ISFdirectory_name)
     featureTable <- ISFrag::ms2.assignment(MS2directory = ISFdirectory_name, customFT = customFT)
     featureTable <- featureTable[,-1]
     level3 <- ISFrag::find.level3(MS1directory = ISFdirectory_name,
                                   MS1.files = mzMLfile[1],
                                   featureTable = featureTable,
                                   type = "single")
     level2 <- ISFrag::find.level2(ISFtable = level3)
     level1 <- ISFrag::find.level1(ISF_putative = level2)
     results <- ISFrag::get.ISFrag.results(ISF_List = level1, featureTable = featureTable)
     result <- results$FeatureTable
     isf_featuerTable <- cbind(customFT[,1],result)
     colnames(isf_featuerTable)[1] <- "featureID"
     col_index <- which( colnames(isf_featuerTable) %in% c(colnames(peaks), "cl" ,"ISF_level"))
     result <- isf_featuerTable[,col_index]
     result <- result[result$cl != 0,]
     file.remove(paste0(ISFdirectory_name, "/",mzMLfile[1]))
     write.csv(result, paste0(ISFdirectory_name,"/isf_results.csv"), row.names = FALSE)
   }else{result <- 0}
* Identify salt adducts, isotopes

  ```
  xcmsrawlcms <- eicRawlcms(mzMLdirectory = mzmldir, mzMLfile = mzMLfile[1]) # pre-process the raw lcms for EIC extracion
  cl_tb_POS <- cl_tb[cl_tb$cl !=0,]
  cl_tb_cleaned <- ChloroDBPFinder::cleanFeature(peaks = peaks, chlorine_tb = cl_tb_POS,
                                 rawlcms = xcmsrawlcms, rawfile_dir = mzmldir, lcmsfile = mzMLfile[i],
                                 adducts = TRUE, isotopes = TRUE, inSourceFrag = isfrag,
                                 ISFtable = result,
                                 samNum = i)
  ```
* Output the table of chlorine-containing features
  ```
  write.csv(cl_tb_cleaned, paste0(mzmldir,"/",strsplit(mzMLfile[1], split = lcmspattern)[[1]][1],"_",nrow(cl_tb_cleaned), "_cl.csv"),row.names = FALSE )
  high_quality_cl_tb <- cl_tb_cleaned[cl_tb_cleaned$Adduct == 0 & cl_tb_cleaned$isotope == 0 & cl_tb_cleaned$ISF_level == 0,]
  write.csv(high_quality_cl_tb, paste0(mzmldir,"/",strsplit(mzMLfile[1], split = lcmspattern)[[1]][1], "_", nrow(high_quality_cl_tb), "_cl_high_quality.csv"), row.names = FALSE)
  ```
  
### Part 2 Alignment across samples
* 
  ```angular2html
  # Alignment across samples  
  aligned_tb <- alignFeature(file_dir = mzmldir, filePattern = "_cl_high_quality.csv")
  # Output the table of aligned Cl-containing features
  write.csv(aligned_tb, paste0(mzmldir, "/",nrow(aligned_tb),"_alignment.csv"), row.names = FALSE)

    ```
  - file_dir: the path of the folder containing all the feature tables
  - filePattern: the pattern of the feature table name, e.g. "_cl_high_quality.csv"
  - align_mz_tol: m/z tolerance for alignment, default 0.01 Da.
  - align_rt_tol: retention time tolerance for alignment, default 20 seconds.

### Part 3 Missing value imputation 
*
  ```angular2html
  # Restore the missing values
  filled_tb <- fillGap(file_dir = mzmldir, mzmlfiles_pattern = lcmspattern, aligned_tb = aligned_tb )
  # Output
  write.csv(filled_tb, paste0(mzmldir, "/gap_filled.csv"), row.names = FALSE )
    ```
    - file_dir: the path of the folder containing all the mzML files
    - mzmlfiles_pattern: the pattern of the mzML file name, e.g. ".mzML" or ".mzXML"
    - aligned_tb: aligned feature table.
    - int_threshold: intensity threshold for missing value imputation, default 0.
  
### Part 4 Annotation
  * Specify the path feature table that needs to annotate. <br>
    Users need to manually change the path within read.csv() function below
    ```angular2html
    table_need_annotation <- read.csv("C:/Users/User/Desktop/testmzML/TW APM noAscorbic_159_cl_high_quality.csv")
    ```
    
  * Spectral database search
    ```angular2html
    # Load database
    Cl_db <-  read.csv(Cl_db_path)
    # Compound annotation by spectral database search
    annotated_tb <- annonateFeature(featureTable = table_need_annotation, Cl_db, ion_mode = "P", ref_mz_tol =25, dp_score = 70, dp_num = 2)
    # Output annotations
    write.csv(annotated_tb, paste0(mzmldir, "/", nrow(annotated_tb[annotated_tb$score!=0,]),"_annotations.csv"), row.names = FALSE)

    ```
    - table_need_annotation: a feature table that needs annotation
    - ion_mode: "P" for positive mode, "N" for negative mode
    - ref_mz_tol: m/z tolerance for spectral database search
    - dp_score: score threshold, default 70 out of 100.
    - dp_num: number threshold of matched fragments, default 2.

  * Molecular networking
    ```angular2html
    # Load the reference table containing known compounds or precursors
    reference_table <- read.csv(ref_path)
    
    # load reaction pathway
    data("reaction_pathways")
    
    # construct the network
    network <- createNetwork(featureTable = annotated_tb , seed_tb = reference_table,
                         reaction_pathways = reaction_pathways, reaction_pathway_mz_tol = 0.01,
                         nw_spectra_score=0.5, nw_spectra_match_num=3)
    ```
    - nw_spectra_score: spectral similarity score to construct spectral network, default 0.5 out of 1
    - nw_spectra_match_num: the number of matched fragment for spectral network, default 3.
    - reaction_pathway_mz_tol: the m/z tolerance for reaction network, default 0.01 Da.
    
    ```angular2html
    # Output reaction and spectral networks
    all_nw <- network[[1]]
    write.csv(all_nw, paste0(mzmldir, "/molecular_network.csv"), row.names = FALSE)
    
    # Output network with explainable connections which have high spectral connection and reaction pathway connection
    integrated_nw <- network[[2]]
    write.csv(integrated_nw, paste0(mzmldir, "/integrated_molecular_network.csv"), row.names = FALSE)
    ```
   
    
## Citation
If you use ChloroDBPFinder in your research, please cite the following paper:
 


