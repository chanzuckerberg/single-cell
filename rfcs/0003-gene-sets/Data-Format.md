# Gene Sets Data Format

**Authors:** [Madison Dunitz](mailto:madison.dunitz@chanzuckerberg.com)

**Approvers:** [Arathi Mani](mailto:arathi.mani@chanzuckerberg.com), [Signe Chambers](mailto:schambers@chanzuckerberg.com), [Friendly Tech Person2](mailto:colin.megill@chanzuckerberg.com), [Girish Patangay](mailto:girish.patangay@chanzuckerberg.com), [Sara Gerber](mailto:sara.gerber@chanzuckerberg.com)

## tl;dr

This RFC contains the details about the data format for the gene set feature. It will support the import and export of differential
expression results and allow a user to upload their own gene sets for analysis.

## Glossary

- Gene set - List of genes
- Differential expression - A statistic (typically logfold and p-value) describing the difference in gene expression between two sets of cells. It may refer to stats specific to one gene (differential expression value) or many (differential expression analysis). When referring to many genes it is typically capped (eg top 10 most differentially expressed genes)
- LogFold - A ratio of the difference in expression levels. Typically calculated by taking the log2 of the gene expression value for each cell and then finding the mean of that value for each set of cells. Because the data is already log2 transformed you can then take the difference between the two means as the logfold value

## Problem Statement | Background

Gene sets are the expected output of differential expression analysis in cellxgene. Two (non-overlaping) sets of cells
are selected and the difference in the expression level for each gene is computed by comparing the average
expression level for each set of cells. When there is a particularly large difference in expression between the two sets of
cells, that gene is identified and presented to the user. To ensure consistency and compatibility it is necessary to define a
data format for storing references to these genes and any accompanying statistical data.

## Product Requirements

### Users should be able to export differential expression results

Currently cellxgene users can choose two sets of cells and compute/display the log fold and adjusted p-value of the top ten differentially expressed genes. Additionally we display a histogram of the gene expression levels of the selected cells for each gene in the right side bar.

- [US1] A user should be able to export a gene set, if the gene set was created by differential expression, the pvalue, log fold change, and category label fields will be populated in addition to the gene name. This will be available in a csv file (see data formats below for more detail).

### Users should be able to upload their own gene sets and see them in the right side bar with the option to apply them within current cellxgene functionalities, if the same gene ontology was used when generating the current dataset and the gene set uploaded by the user (ie the gene names match)

- [US2] A user should be able to upload a gene set via a csv file (see data formats below for more detail) for a dataset.
- [US3] A user should be able to upload a gene set they previously downloaded from cellxgene
- [US4] When a user uploads a gene set, that list of genes should appear in the right side bar [Note the scope of this RFC does not include how the upload process will work]
- [US5] If the user was logged in when they uploaded a gene set that gene set should persist if they logout and back in or reload the page
- [US6] If the gene names match those used in the dataset the user should be able to use general cellxgene functionality
  (color by genes, calculate differential expression levels, etc.) on those genes.
- [US7] If a gene in the user uploaded gene set does not exist in the dataset the user will see an error message specifying the gene that wasnt in the dataset, but the remaining genes will be displayed.

## Detailed Design | Architecture | Implementation

### Data model

Fully supporting persistence of differential expression results is out of the scope of this RFC. But the data model described below has been designed to allow for differential expression persistence in the future.

### Relational Database

To support gene sets two tables will be added to the cellxgene relational database

- Gene
  - UUID (generated)
  - GeneName
  - Comments
  - GeneSet
  - created_at
  - updated_at
- GeneSet
  - UUID (generated)
  - ConsensusCounter (generated)
  - GeneSet Name (required)
  - Gene Count (generated)
  - User UUID
  - Dataset UUID
  - comments
  - created_at
  - updated_at

![Cellxgene Data Schema](imgs/Cellxgene_rds_schema.png)
**Figure 1** Cellxgene data schema, tables to be added (as described above) are in red

#### CSV Format

Gene Set CSV

- This CSV can contain multiple gene sets.

- A file containing a header listing the fields (`GENESET,GENE,COMMENTS,PVALUE,LOGFOLD`)
  and a comma separated list of the gene set name, genes, comments, pvalue, and logfold value,
  identifiers for the cell sets that were compared to produce those values and details about why a particular gene was chosen will
  be included in the comments value (eg tissue.lung vs. tissue.heart or I chose this gene because ...). Each gene should
  be on a new line. Only gene set (name) and gene are required fields. The name of the file will be based on the user
  name and a timestamp rounded to the nearest second. If cells from two different datasets are being compared the
  datasest name should be appended to the cellset label in the comments section
  (eg dataset1.categoryA.label1 vs dataset2.categoryA.label2).

```CSV
GENESET,GENE,COMMENTS,PVALUE,LOGFOLD
SET1, a23, I picked this gene because I think it causes cancer,,
SET1, b46, I picked this gene because I think it prevents cancer,,
SET1, c19, I picked this gene because it has a funny name,,
SET2, d57, tissue.lung vs. tissue.heart, .05, 2
SET2, d48, tissue.lung vs. tissue.heart, .01, 4
SET2, d89, tissue.lung vs. tissue.heart, .35, 2
```

## Alternatives

Because of the redundancy of some pieces of data in the csv (gene set name, cellset identifiers in the comments) we
considered using a json format instead, however discussions with scientists (our main users) made it clear that they
felt more comfortable editing and working with a csv file.

## References
