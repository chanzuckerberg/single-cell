# Gene Sets Data Format

## [Optional] Subtitle of my design 

**Authors:** [Madison Dunitz](mailto:madison.dunitz@chanzuckerberg.com)

**Approvers:** [Friendly Tech Person](mailto:some.nice.person@chanzuckerberg.com), [Friendly PM Person](mailto:some.nice.person@chanzuckerberg.com), [Friendly Tech Person2](mailto:some.nice.person@chanzuckerberg.com), [Girish Patangay](mailto:girish.patangay@chanzuckerberg.com), [Sara Gerber](mailto:sara.gerber@chanzuckerberg.com) 

## tl;dr 

This RFC contains the details about the data format for the gene set feature. It will support the export of differential
 expression results and allow a user to upload their own gene sets for analysis. 
 
## Glossery 
- Gene set - List of genes
- Differential expression - A statistic (typically logfold and p-value) describing the difference in gene expression between two sets of cells. It may refer to stats specific to one gene (differential expression value) or many (differential expression analysis). When referring to many genes it is typically capped (eg top 10 most differentially expressed genes)
- LogFold - A ratio of the difference in expression levels. Typically calculated by taking the log2 of the gene expression value for each cell and then finding the mean of that value for each set of cells. Because the data is already log2 transformed you can then take the difference between the two means as the logfold value 
## Problem Statement | Background

Gene sets are the expected output of differential expression analysis in cellxgene. Two (non-overlaping) sets of cells 
are selected and the difference in the expression level for each gene is computed by comparing the average 
expression level for each set of cells. When there is a particularly large difference in expression between the two sets of 
cells, that gene is identified and presented to the user. To ensure consistency and compatibility it is necessary to define a 
data format for storing references to these genes and any accompanying statistical data. 

## [Optional] Product Requirements
### Users should be able to export differential expression results
Currently cellxgene users can choose two sets of cells and compute/display the log fold and adjusted p-value of the top ten differentially expressed genes. Additionally we display a histogram of the gene expression levels of the selected cells for each gene in the right side bar. 
- [US1] A user should be able to export the list of 10 genes along with their logfold and adjusted p-values in a csv file (see data formats below for more detail) by clicking a button
### Users should be able to upload their own gene sets and see them in the right side bar with the option to apply them within current cellxgene functionalities, if the same gene ontology was used when generating the current dataset and the geneset uploaded by the user (ie the gene names match) 
- [US2] A user should be able to upload a geneset via a csv file (see data formats below for more detail) for a dataset.
- [US2.5] A user should be able to upload a geneset they previously downloaded from cellxgene
- [UC3] When a user uploads a geneset, that list of genes should appear in the right side bar 
- [US4] If the user was logged in when they uploaded a geneset that genesit should persist if they logout and back in or reload the page
- [US5] If the gene names match those used in the dataset the user should be able to use general cellxgene functionality (color by genes, calculate differential expression levels, etc.) on those genes
- [US6] Genesets uploaded for one dataset should be available in other datasets?

### If there is a standard format for differential expression output, we should follow that. If not, the format should be as straightforward as possible while serving these above cases.

### [Optional] Nonfunctional requirements
- Users should be able to export precomputed differential expression results in real time
- Users should be able to upload gene sets and have them immediately appear in the right side bar

## Detailed Design | Architecture | Implementation


### [Optional] Data model

Fully supporting persistence of differential expression results is out of the scope of this RFC. But the data model described below has been designed to allow for differential expression persistence in the future.
###Relational Database
To support gene sets five tables will be added to the cellxgene relational database
- Gene
   - UUID (generated)
   - GeneName
   - OntologyId?
- GeneGeneSetLink
   - UUID
   - Gene UUID
   - GeneSet UUID
   - Comments (optional)
- GeneSet
   - UUID (generated)
   - GeneSet Name (required)
   - Gene Count (generated)
- GeneSetUserLink
   - User UUID
   - GeneSet UUID
   - Comments (optional)
   - Dataset UUID?
-GeneSetDatasetLink
   - UUID (generated)
   - GeneSet UUID
   - Dataset UUID

Note - GeneSets will only be linked to datasets when they were uploaded with that dataset
![Cellxgene Data Schema](imgs/Cellxgene_rds_schema.png)
**Figure 1** Cellxgene data schema, tables to be added (as described above) are in red
#### Imported CSV
GeneSet CSV
- A file containing a comma separated list of the gene set name, genes, pvalue, logfold value, 
identifers for the cellsets that were compared to produce those values and details about why they were included. 
Each gene should be on a new line. Only gene set (name) and gene are required fields. The name of the file will be based on the 
user name and a timestamp rounded to the nearest second. If cells from two different datasets are being compared the datasest name 
should be appended to the cellset label (eg dataset1.categoryA.label1)
```
GENESET,GENE,PVALUE,LOGFOLD,CELLSET1_CATEGORY.LABEL,CELLSET2_CATEGORY.LABEL,COMMENTS
SET1, a23, .05, 2, tissue.lung, tissue.heart, I picked this gene because I think it causes cancer
SET1, b46, .01, 4, tissue.lung, tissue.heart, I picked this gene because I think it prevents cancer
SET1, c19, .35, 2, tissue.lung, tissue.heart, I picked this gene because it has a funny name
SET2, d57, .09, 3, cluster.1, cluster.2,
SET2, d48, .04, 4, cluster.1, cluster.2 
SET2, d89, .06, 5, cluster.1, cluster.2 
```

## [Optional] Alternatives

Because of the redundancy of the some pieces of data in the csv (geneset name, cellset identifiers) we considered using a json format instead,
however discussions with scientists (our main users) made it clear that they felt more comfortable editing and working with a csv file.

## References
