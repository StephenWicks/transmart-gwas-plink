# transmart-gwas-plink
Plink integration for tranSMART

Installation
------------

Add the following entry into "inlinePlugins":

  'transmart-gwas-plink': 'transmart-gwas-plink'

and the following entry into "InternalDependenciesFilter":

  compile ":transmart-gwas-plink:$transmartVersion"

in file "DependencyManagement.groovy"

Add the following lines into Config.groovy:

  grails.plugin.transmartGwasPlink.enabled=true
  grails.plugin.transmartGwasPlink.plinkPath='/usr/local/bin/plink'

(the value of the last parameter depends on where you've installed plink binary).

Usage
-----
#GWAS Documentation
##Introduction.

The PLINK application is commonly used to identify in a genome-wide way, alleles that are relatively enriched or depleted in a population (relative to a control population) of individuals with a particular trait. This trait is usually operationally defined as one state of a binary variable (commonly a disease state) but the application can also be run as a linear regression with a data variable structured as a quantitative measure (for example, height). 
The source of data for this GWAS analysis can either be derived from GWAS-specific platforms (so called “SNP” chips, though note that “SNP” data is often loosely used to also describe copy-number (CNV/LoH) data; the SNP data here refers to subject level polymorphism identification as might be obtained from a VCF file), or from general sequencing of large populations. The actionable GWAS data is the allelic representation of many (hundreds of thousands to millions) random (or intentionally dispersed in such a way as to maximize information content) SNPs from across the genome of many (hundreds to tens or hundreds of thousands) individuals. Once a SNP dataset has been acquired, it can be used in many distinct analyses, either related to the primary endpoint, or completely unrelated, as long as reliable phenotyping for traits of interest is available. Consequently, loading a SNP dataset with a large amount of patient-indexed clinical data has great value in facilitating GWAS data analysis.

GWAS data is stored and analyzed in a largely binary format defined by the PLINK application (Although, ASCII text files are PLINK compatible, the file sizes and processing rate are so advantaged with binary datasets, that current implementations of PLINK will convert the files to binary format before executing an association test. As such, these data are loaded into the tranSMART database directly as binary files.) The set of three files that filly describe a PLINK dataset are:
The BED file: This file describes the genotypes of a population of individuals. Each row represents a patient, and each column represents a single SNP value.

The BIM file: This file is a SNP-indexed platform file that contains information about every SNP in that is represented in the BED file.
The FAM file: This text file is a small file containing service information on each subject, that may include the subjectID, familyID, paternal and maternal IDs, gender, and the target variable (case/control binary, or continuous scalar).

##The tranSMART implementation: data loading
The GWAS functionality of tranSMART is straightforward, and under active development. A basic functionality is available now, and we hope to be able to provide access to a larger set of parameters in the future. 
The GWAS data analysis package ‘PLINK’ should be installed and running on the tranSMART application server. The server should be provisioned with significant available storage for the temporary storage of PLINK results that can be downloaded after an analysis is invoked (and appropriate trash collection jobs be scheduled on the server to clean up older results sets). A table has been created that contains GWAS PLINK data in binary format if that data is loaded with the ETL tool tmDataLoader. 

      > desc GWAS_PLINK.PLINK_DATA
      Name          Null?    Type
      ------------- -------- ------------
      PLINK_DATA_ID NOT NULL NUMBER(10)
      STUDY_ID      NOT NULL VARCHAR2(50)
      BED           BLOB
      BIM           BLOB
      FAM           BLOB

The data is stored compressed using LZO algorithm. Every time you launch an analysis, these files are extracted from the database into the application server. PLINK is run locally on the application server, and results are stored and made available for download from there.
To load PLINK Data you should have the following folders and files in your study directory:

    <STUDY_NAME>_<STUDY_ID>
        \GWASPlinkData\ (or \GWASPlinkDataToUpload)
          MappingFile.txt
          <ID>.bed
          <ID>.bim
          <ID>.fam
      ...
    ...
The mapping file should contain the following lines:
    # STUDY_ID: <STUDY_ID>
    # BFILE: <ID>
    # CATEGORY_CD: <PATH for PLINK DATA IN TREE>

The <ID> in the “BFILE:” line corresponds to the “ID that prefixes the .bed, .bim, and .fam files. 
#An association test
When you want to run an association test, simple create two non-overlapping cohorts. This may be done using a single categorical variable, or may be driven by complex subset formation using a variety of clinical or high dimensional variables. The “.fam” file associated with that particular invocation of PLINK is re-written to reflect inclusion of subjects in either cohort 1 and cohort 2 from the subset selection window. The PLINK “--keep” flag is applied to all analyses, to ensure that the source data is reduced to include only relevant subjects. Once cohorts are formed, select the GWAS Plink option from the “Analysis” pull-down menu of the “Advanced Workflow” tab. Ensure the “Association analysis” pull-down option is selected. The “Result Name” (default: “result” field will be used to differentiate result sets. The “Phenotypes” toggle and variable selection box are currently not to be used (in development) for this kind of analysis. 
#Logistic and Linear regression
You can also run a logistic regression based analysis (which also allows the selection of a covariate, such as Race or Sex, that might vary systematically in case or control populations; currently, only one covariate is supported), by selecting Logistic Regression from the pull-down menu. Cohorts for a logistic regression are generated as for the Association test described above.
If a quantitative variable is used, rather than two cohorts, to drive an analysis using linear regression, the user should specify one cohort of interest rather than two. The numeric variable to regress against should be dragged to the “phenotype” box. A new file (<ID>.pheno.txt) will be created, and used with the PLINK “--pheno” option to drive the linear regression analysis. As is the case with logistic regression, covariates can be specified in a linear regression if desired. 
#Results:
The output of an association test will be limited to the set of results below the unadjusted p-value specified and the number of results specified in the options. The table shown in the result pane shows the SNP list sorted by pval, and lists the chromosome, SNPid, unadjusted pval, and a variety of corrected significance tests. The full table can also be obtained by clicking the “Download PLINK results” link at the bottom of the page. This link also gives the “result.assoc” file, wherein the following information can be found:

         CHR     Chromosome
         SNP     SNP ID
         BP      Physical position (base-pair)
         A1      Minor allele name (based on whole sample)
         F_A     Frequency of this allele in cases
         F_U     Frequency of this allele in controls
         A2      Major allele name
         P       Exact p-value for this test
         OR      Estimated odds ratio (for A1)

