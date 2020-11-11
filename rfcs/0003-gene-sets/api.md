# Gene Set Data Format

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

### Users should be able to export differential expression results

Currently cellxgene users can choose two sets of cells and compute/display the log fold and adjusted p-value of the top ten differentially expressed genes. Additionally we display a histogram of the gene expression levels of the selected cells for each gene in the right side bar.

- [US1] A user should be able to export the list of 10 genes along with their logfold and adjusted p-values in a csv file (see data formats below for more detail) by clicking a button

### Users should be able to upload their own gene sets and see them in the right side bar with the option to apply them within current cellxgene functionalities, if the same gene ontology was used when generating the current dataset and the geneset uploaded by the user (ie the gene names match)

- [US2] A user should be able to upload a geneset via a csv file (see data formats below for more detail) for a dataset
- [UC3] When a user uploads a geneset, that list of genes should appear in the right side bar
- [US4] If the user was logged in when they uploaded a geneset that genesit should persist if they logout and back in or reload the page
- [US5] If the gene names match those used in the dataset the user should be able to use general cellxgene functionality (color by genes, calculate differential expression levels, etc.) on those genes
- [US6] Genesets uploaded for one dataset should be available in other datasets?

### If there is a standard format for differential expression output, we should follow that. If not, the format should be as straightforward as possible while serving these above cases

### [Optional] Nonfunctional requirements

Nonfunctional requirements are not directly related to the user story but may instead reflect some technical aspect of the implementation, such as latency. [Here's](https://en.wikipedia.org/wiki/Non-functional_requirement) a more comprehensive list of nonfunctional requirements.

- Users should be able to export precomputed differential expression results in real time
- Users should be able to upload gene sets and have them immediately appear in the right side bar

## Detailed Design | Architecture | Implementation

Because genesets are closely tied to differential expression, some aspects of the differential expression design plans will also be detailed below.

### User uploaded gene sets

#### 1. User creates csv containing a list of genes (and optional comments about why each gene was included)

CSV must follow format described below in the Data Model section

With a cap of 500 genes per set this file should never be larger than 50kb

#### 2. User uploads file to hosted cellxgene client

This will typically be from their local machine but we should consider supporting allowing uploads from s3 in future iterations.

The user does not need to be logged in to upload a geneset, but they will need to be logged in if they want that geneset to persist to future sessions.

Users will upload the file by passing its path to the client.
The client will validate the format of the csv

- if the format is invalid the client will raise a descriptive error message to the user
- if the file is valid, the client will display the genes in the right side bar

If the user is logged in the frontend will send a post request to the /gene_set/upload (see API description below) with the geneset csv as the body of the request and the file name as the name of the geneset

The backend will parse the csv and create a geneset, linking it to the user via the GeneSetUserLink.
Genes listed in the geneset will be created (if they dont exist) in the gene table and linked to the geneset via the GeneGeneSetLink, comments included in the csv will be stored as comments on in the GeneGeneSetLink

- If the geneset is successfully stored by the backend will respond with a 200.
- If it is not successfully saved the backend will respond with a 400.

No action will be taken by the client if the geneset is successfully saved

If the backend returns a 400 the client will alert the user that their geneset has not been saved.

### Export Differential Expression Output

#### Current Functionality

The user selects a set of cells via the lasso and then clicks the 1:0 cells button shown in Figure 2

The user selects a set of cells via the lasso and then clicks the 2:0 cells button shown in Figure 2

Once two sets of cells have been selected the button to the right of 2:0 cells (diffexp button) becomes available as shown in Figure 3. Figure 3 also shows that the cell selection button is updated to show the number of cells currently selected.

When the user selects the diffexp button, the client calculates the difference in expression levels for all of genes between the two sets of cells. It then display the 10 most differentially expressed genes in the right side bar along with a histogram of their expression levels.

![Cellxgene Data Schema](imgs/diff_expression_button_pre.png)

**Figure 2** Cellxgene Diff Expression button pre cell selection

![Cellxgene Data Schema](imgs/diff_expression_button_post.png)

**Figure 3** Cellxgene Diff Expression button post cell selection

#### Additional Functionality

In the future we will limit the user's ability to create anonymous groups of cells. They will need to use the annotation feature to create a category and label the cell group in order to select it for differential expression analysis.

The user will be able to export a CSV file containing the 10 most differentially expressed genes along with their logfold and p-values.

The csv file will be named by the category, label of the two sets of cells being compared and be stored under that dataset identifier. See below in data structures for more detail.

### APIs

#### POST geneset/upload

An authenticated user can upload a csv containing a set of genes.

**Request:**

| Parameter    | Description                                                 |
| ------------ | ----------------------------------------------------------- |
| geneset_name | Identifies the geneset to be created.                       |
| body         | Contains the csv geneset with any accompanying descriptions |

**Response:**

| Code | Description                        |
| ---- | ---------------------------------- |
| 200  | The geneset was successfully saved |

| Key         | Description                                   |
| ----------- | --------------------------------------------- |
| status_code | Provides the current status of the upload.    |
| message     | If an error occurred, the message shows here. |

**Error Responses:**

| Code | Description                             |
| ---- | --------------------------------------- |
| 400  | If the parameters supplied are invalid. |

### [Optional] Data model

Fully supporting persistence of differential expression results is out of the scope of this RFC. But the data model described below has been designed to allow for differential expression persistence in the future.

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

![Cellxgene Data Schema](imgs/Cellxgene_rds_schema.png)
**Figure 1** Cellxgene data schema, tables to be added (as described above) are in red

GeneSet CSV

- A file containing a comma separated list of genes and (optionally) details about why they were included. Each gene should be on a new line The name of the file will be used as the name of the geneset (see dataset_id.csv for example)

Exported File

- A file containing a comma separated list of genes, logfold values and p-values. The name of the directory storing the file will reference the dataset and the name of the file will contain names of the cell sets being compared, see datasetID/categoryA.label1-categoryB.label2.csv for an examole
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

[0][image of random system architecture](https://www.visual-paradigm.com/features/aws-architecture-diagram-tool/).

[1][google developer documentation style guide](https://developers.google.com/style/highlights#introduction_1)
