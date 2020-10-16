# Gene Set Data Format

The data format for gene sets should also support the export of differential expression results and a user uploading 
their own distribution for a gene set. If there is a standard format for differential expression output, we should 
follow that. If not, the format should be as straightforward as possible while serving these use cases.




## [Optional] Subtitle of my design 

**Authors:** [Nice Person](mailto:some.nice.person@chanzuckerberg.com)

**Approvers:** [Friendly Tech Person](mailto:some.nice.person@chanzuckerberg.com), [Friendly PM Person](mailto:some.nice.person@chanzuckerberg.com), [Friendly Tech Person2](mailto:some.nice.person@chanzuckerberg.com), [Girish Patangay](mailto:girish.patangay@chanzuckerberg.com), [Sara Gerber](mailto:sara.gerber@chanzuckerberg.com) 

## tl;dr 

This RFC contains the details about the data format for the gene set feature. It will support the export of differential
 expression results and allow a user to upload their own gene sets for analysis. 
 
## Glossery 
- Geneset - List of genes
- Differential expression - A statistic (typically logfold and p-value) describing the difference in gene expression between two sets of cells. It may refer to stats specific to one gene (differential expression value) or many (differential expression analysis). When referring to many genes it is typically capped (eg top 10 most differentially expressed genes)
- LogFold - A ratio of the difference in expression levels. Typically calculated by taking the log2 of the gene expression value for each cell and then finding the mean of that value for each set of cells. Because the data is already log2 transformed you can then take the difference between the two means as the logfold value 
## Problem Statement | Background

Gene sets are the expected output of differential expression analysis in cellxgene. Two (non-overlaping) sets of cells 
are selected and the difference in the expression level for each gene is computed by comparing the average 
expression level for each set of cells. When there is a particularly large difference in expression between the two sets of 
cells, that gene is identified and presented to the user. To ensure consistency and compatibility it is necessary to define a 
data format for storing references to these genes and any accompanying statistical data. 

## [Optional] Product Requirements

This section should come primarily from a Product Manager. For large designs, technical designs should be closely aligned with a product goal and thus should be explicitly bullet-pointed in this section to ensure alignment.
### Users should be able to export differential expression results
Currently cellxgene users can choose two sets of cells and compute/display the log fold and adjusted p-value of the top ten differentially expressed genes. Additionally we display a histogram of the gene expression levels of the selected cells for each gene in the right side bar. 
- [US1] A user should be able to export the list of 10 genes along with their logfold and adjusted p-values in a csv file (see data formats below for more detail) by clicking a button
### Users should be able to upload their own gene sets and see them in the right side bar with the option to apply them within current cellxgene functionalities, if the same gene ontology was used when generating the current dataset and the geneset uploaded by the user (ie the gene names match) 
- [US2]  A user should be able to upload a geneset via a csv file (see data formats below for more detail) for a dataset
- [UC3] When a user uploads a geneset, that list of genes should appear in the right side bar 
- [US4] If the user was logged in when they uploaded a geneset that genesit should persist if they logout and back in or reload the page
- [US5] If the gene names match those used in the dataset the user should be able to use general cellxgene functionality (color by genes, calculate differential expression levels, etc.) on those genes
- [US6] Genesets uploaded for one dataset should be available in other datasets?

### If there is a standard format for differential expression output, we should follow that. If not, the format should be as straightforward as possible while serving these above cases.

### [Optional] Nonfunctional requirements

Nonfunctional requirements are not directly related to the user story but may instead reflect some technical aspect of the implementation, such as latency. [Here's](https://en.wikipedia.org/wiki/Non-functional_requirement) a more comprehensive list of nonfunctional requirements.
- Users should be able to export precomputed differential expression results in real time
- Users should be able to upload gene sets and have them immediately appear in the right side bar

## Detailed Design | Architecture | Implementation

I leave it up to the author to determine the best layout of this section in order to clearly render their design. Pictures are highly encouraged.

Because genesets are closely tied to differential expression, some aspects of the differential expression design plans will also be detailed below.
### User uploaded gene sets
#### 1. User creates csv containing a list of genes (and optional comments about why each gene was included)
CSV must follow format described below in the Data Model section
#### 2. User uploads file to hosted cellxgene client
This should be 
This will typically be from their local machine but we should consider supporting allowing uploads from s3 in future iterations.

### Export Differential Expression Output

### APIs
![Sample Architecture](imgs/sample_architecture.png)


**Figure 1:** This is the [first Google image](https://www.visual-paradigm.com/features/aws-architecture-diagram-tool/) that came up when I searched “software architecture diagram.”


### [Optional] Data model

This section details anything from database schemas, to filesystem structures, to storage formats. Anything that "persists", is accessed by multiple component, or is something that could cause upgrade issues upon change, should belong in this section.
Fully supporting differential expression is out of the scope of this RFC. But the data model described below has been designed to allow for differential expression in the future.


To support genesets four tables will be added to the cellxgene relational database
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
   - Comments (optional, only available when the geneset is generated by cellxgene)
- GeneSetUserLink
   - User UUID
   - GeneSet UUID
   - Comments (optional)
   - Dataset UUID (?)

<img style="float: right;" src="./imgs/Cellxgene_rds_schema.png" width="200">

GeneSet CSV
- A file containing a comma separated list of genes and (optionally) details about why they were included. Each gene should be on a new line The name of the file will be used as the name of the geneset (see file for example)

Exported File
-  A file containing a comma separated list of genes, logfold values and p-values. The name of the directory storing the file will reference the dataset and the name of the file will contain names of the cell sets being compared, datasetID/categoryA.label1-categoryA.label3 ![Example File](rfcs/XXXX-gene-sets/DatasetID/categoryA.label1_categoryb.label2.csv)
In the future, if cells from two different datasets are being compared the dataset id will be appended as a prefix to the category names in the filename (?will this make the filename too long?)

### [Optional] Test plan

This section details the sorts of tests that will be written to give confidence that the implementation matches the design. Enumerating the test cases is a good way to find holes in the design and helps the reviewers understand, in more detail, how the feature works. It also helps get into the mindset of test driven development.

## Test that an exported list of genes can be reimported

### [Optional] Monitoring and error reporting

This section contains information on the monitoring and error reporting features that will be built in order for the system to respond and alert users on failures of the feature being designed. This helps ensure that failure modes are thought about and the users' experiences with facing errors is consistent and helpful, rather than frustrating.


## [Optional] Alternatives

Especially for larger designs, you probably considered more than one design so it’s worthwhile to briefly list the others and a short note on why you decided against it.


## References

Any relevant documents or prior art should be clearly listed here. Any citation of figures or papers should also be clearly listed here. Of course, they can also be linked throughout the document when directly referenced.

[0] [Image of random system architecture](https://www.visual-paradigm.com/features/aws-architecture-diagram-tool/).

[1] [Google Developer Documentation Style Guide](https://developers.google.com/style/highlights#introduction_1)
