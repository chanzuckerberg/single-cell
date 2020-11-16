# Gene Sets

## API Format

**Authors:** [Madison Dunitz](mailto:madison.dunitz@chanzuckerberg.com)

**Approvers:** [Arathi Mani](mailto:arathi.mani@chanzuckerberg.com), [Signe Chambers](mailto:schambers@chanzuckerberg.com), [Friendly Tech Person2](mailto:colin.megill@chanzuckerberg.com), [Girish Patangay](mailto:girish.patangay@chanzuckerberg.com), [Sara Gerber](mailto:sara.gerber@chanzuckerberg.com)

## tl;dr

This RFC contains the details about the API for the gene set feature. It will support the export of differential
expression results and allow a user to upload their own gene sets for analysis.

## Glossery

- Geneset - List of genes
- Differential expression - A statistic (typically logfold and p-value) describing the difference in gene expression between two sets of cells. It may refer to stats specific to one gene (differential expression value) or many (differential expression analysis). When referring to many genes it is typically capped (eg top 10 most differentially expressed genes)
- LogFold - A ratio of the difference in expression levels. Typically calculated by taking the log2 of the gene expression value for each cell and then finding the mean of that value for each set of cells. Because the data is already log2 transformed you can then take the difference between the two means as the logfold value

## Problem Statement | Background

Gene sets are the expected output of differential expression analysis in cellxgene. Two (non-overlaping) sets of cells
are selected and the difference in the expression level for each gene is computed by comparing the average
expression level for each set of cells. Here I will define the structure of the API to ensure communication between the
client (where the user is working directly with the data or uploading their own gene sets) and the backend (where data
is stored to allow it to persist between sessions) is consistent and clear.

## Product Requirements

### Users should be able to export differential expression results

Currently cellxgene users can choose two sets of cells and compute/display the log fold and adjusted p-value of the top ten differentially expressed genes. Additionally we display a histogram of the gene expression levels of the selected cells for each gene in the right side bar.

- [US1] A user should be able to export a gene set, if the gene set was created by differential expression, the pvalue, log fold change, and category label fields will be populated in addition to the gene name. This will be available in a csv file (see data formats for more detail).

### Users should be able to upload their own gene sets and see them in the right side bar with the option to apply them within current cellxgene functionalities, if the same gene ontology was used when generating the current dataset and the gene set uploaded by the user (ie the gene names match)

- [US2] A user should be able to upload a gene set via a csv file (see data formats for more detail) for a dataset.
- [US3] A user should be able to upload a gene set they previously downloaded from cellxgene
- [US4] When a user uploads a gene set, that list of genes should appear in the right side bar.
- [US5] If the user was logged in when they uploaded a gene set that gene set should persist if they logout and back in or reload the page
- [US6] If the gene names match those used in the dataset the user should be able to use general cellxgene functionality
  (color by genes, calculate differential expression levels, etc.) on those genes.
- [US7] If a gene in the user uploaded gene set does not exist in the dataset the user will see an error message specifying the gene that wasnt in the dataset, but the remaining genes will be displayed.

### If there is a standard format for differential expression output, we should follow that. If not, the format should be as straightforward as possible while serving these above cases

### Nonfunctional requirements

- Users should be able to export precomputed differential expression results in real time
- Users should be able to upload gene sets and have them immediately appear in the right side bar

## Detailed Design | Architecture | Implementation

Because gene sets are closely tied to differential expression, some aspects of the differential expression design plans will also be detailed below.

### User uploaded gene sets

#### 1. User creates csv containing a list of genes (and optional comments about why each gene was included)

CSV must follow format described in the Data Model section

With a cap of 500 genes per set and 100 gene sets per file, this file should never be larger than 500kb

#### 2. User uploads file to hosted cellxgene client

The user will typically upload a file from their local machine but we should consider supporting uploads from s3 in future iterations.

The user does not need to be logged in to upload a gene set, but they will need to be logged in if they want that gene set to persist to future sessions.

Users will upload the file by passing its path to the client.
The client will validate the format of the csv

- if the format is invalid the client will raise a descriptive error message to the user
- if the file is valid, the client will display the genes in the right side bar

If the user is logged in the frontend will send a POST request to /gene_set/ (see API description below) with the gene set csv as the body of the request.

The backend will parse the csv and create a gene set (or sets), linking it to the user via the UserId field.
Genes listed in the gene set will be created in the gene table and linked to the geneset via the GeneSet field. A new row will be created in the
gene table each time the gene is included in a geneset. Comments included in the csv will be stored as comments on the gene unless they are identical for all genes in the set.

- If the gene set is successfully stored the backend will respond with a 200.
- If it is not successfully saved the backend will respond with a 400.

No action will be taken by the client if the gene set is successfully saved

If the backend returns a 400 the client will alert the user that their gene set has not been saved.

### Export Differential Expression Output

#### Current Functionality

The user selects a set of cells via the lasso and then clicks the 1:0 cells button shown in Figure 2

The user selects a set of cells via the lasso and then clicks the 2:0 cells button shown in Figure 2

Once two sets of cells have been selected the button to the right of 2:0 cells (diffexp button) becomes available as shown in Figure 3. Figure 3 also shows that the cell selection button is updated to show the number of cells currently selected.

When the user selects the diffexp button, the client calculates the difference in expression levels for all of genes between the two sets of cells. It then display the 10 most differentially expressed genes in the right side bar along with a histogram of their expression levels.

This information does not currently persist between sessions.

![Cellxgene Data Schema](imgs/diff_expression_button_pre.png)

**Figure 2** Cellxgene Diff Expression button pre cell selection

![Cellxgene Data Schema](imgs/diff_expression_button_post.png)

**Figure 3** Cellxgene Diff Expression button post cell selection

#### Additional Functionality

The gene sets created by a user will persist between sessions. When an authenticated user navigates to a particular dataset
the client will send a GET request to /gene_set/{dataset_id} retrieving all of the user's gene sets for that dataset.

In the future we will limit the user's ability to create anonymous groups of cells. They will need to use the annotation
feature to create a category and label the cell group in order to select it for differential expression analysis.

The user will be able to export a CSV file containing one or all of the genesets they created by selecting cell sets and
calculating the 10 most differentially expressed genes along with their logfold and p-values between those sets.

The comments section of the csv file will specify the two sets of cells being compared to create that gene set.

The name of the exported file will be based on the user name and a timestamp rounded to the nearest second.

If the user was signed in when they created the gene sets, data contained in the exported csv will also be persisted in the
cellxgene database (via the /gene_set POST route), linking the dataset, gene set and user.

### APIs

All APIs assume user identification is available with the request.

#### POST geneset/{dataset_uuid}/upload

An authenticated user can upload a csv containing a set of genes.

**Request:**

| Parameter    | Description                                                  |
| ------------ | ------------------------------------------------------------ |
| dataset_uuid | Identifies the dataset the gene sets are linked to.          |
| body         | Contains the csv gene set with any accompanying descriptions |

**Response:**

| Code | Description                         |
| ---- | ----------------------------------- |
| 200  | The gene set was successfully saved |

| Key     | Description                                   |
| ------- | --------------------------------------------- |
| message | If an error occurred, the message shows here. |

**Error Responses:**

| Code | Description                             |
| ---- | --------------------------------------- |
| 400  | If the parameters supplied are invalid. |

| Code | Description                                                  |
| ---- | ------------------------------------------------------------ |
| 403  | User id invalid, or user does not have access to the dataset |

#### GET geneset/{dataset_uuid}

An authenticated user can retrieve gene sets they uploaded or calculated in a previous session for a given dataset.

**Request:**

| Parameter    | Description                                                 |
| ------------ | ----------------------------------------------------------- |
| dataset_uuid | Identifies the dataset the genesets are linked to.          |
| body         | Contains the csv geneset with any accompanying descriptions |

**Response:**

| Code | Description                        |
| ---- | ---------------------------------- |
| 200  | The geneset was successfully saved |

| Key  | Description                                                              |
| ---- | ------------------------------------------------------------------------ |
| body | Contains a json containing all of the user's gene sets for that dataset. |

**Error Responses:**

| Code | Description                             |
| ---- | --------------------------------------- |
| 400  | If the parameters supplied are invalid. |

## Alternatives

Because of the redundancy of some pieces of data in the csv (gene set name, cellset identifiers in the comments) we
considered using a json format instead, however discussions with scientists (our main users) made it clear that they
felt more comfortable editing and working with a csv file.

## References
