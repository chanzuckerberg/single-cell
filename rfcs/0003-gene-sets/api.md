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

#### 2. User uploads file to hosted cellxgene client

The user will typically upload a file from their local machine but we should consider supporting uploads from s3 in future iterations.

The user must be logged in to upload a gene set.

Users will upload the file by passing its path to the client.
The client will validate the format of the csv

- if the format is invalid the client will raise a descriptive error message to the user
- if the file is valid, the client will display the genes in the right side bar

If the user is logged in the frontend will send a POST request to {dataset_name}/api/v0.01/gene_sets/ (see API
description below) with a json object containing the gene sets and genes as the body of the request (see API
description below for more detail about the request body).

The backend will parse the json and create a gene set (or sets), linking it to the user via the UserId field.
Genes listed in the gene set will be created in the gene table and linked to the gene set via the GeneSet field. A new
row will be created in the gene table for each gene included in a gene set.

- If the gene set is successfully stored the backend will respond with a 201 and the uuid and consensus counter for that
  gene set (see keeping consensus below).

- If it is not successfully saved the backend will respond with a 400 and the names of the gene sets that were not created.

No action will be taken by the client if the gene set is successfully saved

If the backend returns a 400 the client will try again and then alert the user that their gene set has not been saved.

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
the client will send a GET request to {dataset_name}/api/v0.01/gene_sets retrieving all of the user's gene sets for
that dataset.

In the future we will limit the user's ability to create anonymous groups of cells. They will need to use the annotation
feature to create a category and label the cell group in order to select it for differential expression analysis.

The user will be able to export a CSV file containing one or all of the genesets they created by selecting cell sets and
calculating the 10 most differentially expressed genes along with their logfold and p-values between those sets.

The comments section of the csv file will specify the two sets of cells being compared to create that gene set.

The name of the exported file will be based on the user name and a timestamp rounded to the nearest second.

### Keeping consensus

This feature will require processing multiple updates to the same gene set, due to our hosting architecture this creates
the potential for race conditions and concerns about keeping the data in sync between the database, server and client.  
To ensure data consistency, when the user is interacting with the data, the client is always the source of truth.
If what is stored in the database doesnt match what the user is seeing, the data in the database will be discarded and
recreated based on the state of the client.

The server will ensure the data is kept in sync through a consensus token or counter. Everytime a gene set is created
in the database it will have a consensus_counter field set to 0000. This will be sent back to the client along with the
gene set name and uuid. When the client updates a gene set (via a PUT request) it will send this counter back to the
server as part of the request body. The server will confirm that the counter sent from the client matches the counter
stored in the database for that gene set.

- If the values match, the server will update the gene set as specified in the PUT
  request and update the counter by incrementing it by 1. The response to the client will include the updated counter value
  to be used in future PUT requests for that gene set.
- If the counter stored in the database does not match the counter sent by the client the server will immediately send
  an error message to the client, specifying the name and uuid of the gene set that is out of sync. The client will
  then send over the entire gene set including all linked genes as a POST request. Because the gene set name is already
  taken for that user/dataset, the server will delete the currently stored gene set and any associated genes and recreate
  it based on the information included in the POST request. This will generate a new UUID for the gene set, and reset the
  consensus counter.
  In the case that the server times out or does not respond to a PUT request from the client, the client can retry or
  just send a POST request to recreate the gene set.
  Because recreating the gene set assigns it a new UUID, any pending or high latency PUT requests sent prior to the update
  be ignored because they reference a UUID that no longer exists.

### APIs

All APIs assume user identification is available with the request and the dataset being worked on is specified in the url
Future API endpoints being considered

- LIST dataset_name/api/v0.1/gene_sets/ - all genesets for a dataset (just names)
- GET dataset_name/api/v0.1/gene_sets/{gene_set_uuid} - get all information for one particular gene set

#### Get dataset_name/api/v0.1/genesets/

Return all of the gene sets (and accompanying genes) associated with a given user/dataset

**Request:**
| Parameter | Description |
| ------------ | --------------------------------------------------------------------------------------------- |
| dataset_name | Identifies the dataset the gene sets are linked to. |
| user_id | Identifies the user creating the gene set |

**Response:**

| Code | Description                              |
| ---- | ---------------------------------------- |
| 200  | The gene sets were successfully returned |

| Key           | Description                                                                         |
| ------------- | ----------------------------------------------------------------------------------- |
| response_body | List of gene set dicts containing their identifying information and a list of genes |
|               | {"gene_sets":                                                                       |
|               | [{                                                                                  |
|               | "gene_set_name":"gene set name",                                                    |
|               | "gene_set_uuid": "uuid".                                                            |
|               | "consensus_counter": "0001"                                                         |
|               | "comments": "[OPTIONAL]Comments describing gene set",                               |
|               | "genes":                                                                            |
|               | [{                                                                                  |
|               | "gene name": "[Required] name of gene",                                             |
|               | "comments": "[OPTIONAL] comments on why a gene was included in this gene set"       |
|               | }]                                                                                  |
|               | }]                                                                                  |
|               | }                                                                                   |

**Error Responses:**

| Code | Description                             |
| ---- | --------------------------------------- |
| 400  | If the parameters supplied are invalid. |

| Code | Description                                                  |
| ---- | ------------------------------------------------------------ |
| 403  | User id invalid, or user does not have access to the dataset |

#### POST dataset_name/api/v0.1/gene_sets/

Any gene sets created or uploaded (via the csv format described above) will be sent via json to the backend for longterm
storage/persistence.
Gene sets and gene names must be unique for a user/dataset - if an already used gene set name is submitted, the backend
will assume it is due to a consensus issue and delete the currently stored gene set (and any attached genes) and
recreate it based on the information in the request

**Request:**
| Parameter | Description |
| ------------ | --------------------------------------------------------------------------------------------- |
| dataset_name | Identifies the dataset the gene sets are linked to. |
| user_id | Identifies the user creating the gene set |  
| body | JSON formatted dict containing a list of gensets (and accompanying genes) to be created |
| | {"gene_sets": |
| | [{ |
| | "gene_set_name":"[Required] gene set name", |
| | "comments": "[OPTIONAL]Comments describing gene set", |
| | "genes": |
| | [{ |
| | "gene name": "[Required] name of gene", |
| | "comments": "[OPTIONAL] comments on why a gene was included in this gene set" |
| | }] |
| | }] |
| | }  
**Response:**

| Code | Description                         |
| ---- | ----------------------------------- |
| 201  | The gene set was successfully saved |

| Key       | Description                                                                               |
| --------- | ----------------------------------------------------------------------------------------- |
| gene_sets | Dict mapping the gene set name to its assigned uuid and its consensus counter             |
|           | {"name_of_gene_set": {"uuid": "gene_set_uuid", "consensus_counter": "0001"}}              |
| errors    | In the case where some gene sets were successfully stored and others will not, a message  |
|           | specifying the error and the gene set(s) it was linked to is returned to the client here. |

**Error Responses:**

| Code | Description                             |
| ---- | --------------------------------------- |
| 400  | If the parameters supplied are invalid. |

| Code | Description                                                  |
| ---- | ------------------------------------------------------------ |
| 403  | User id invalid, or user does not have access to the dataset |

#### PUT dataset_name/api/v0.1/gene_sets/{gene_set_uuid}

Any changes to a gene set (the gene set itself or genes within that gene set) will be sent via json to the backend.
This includes the creation/deletion of genes within a gene set
Updates to comments will overwrite the currently stored comments
**Request:**

| Parameter     | Description                                                                                      |
| ------------- | ------------------------------------------------------------------------------------------------ |
| dataset_name  | Identifies the dataset the gene sets are linked to.                                              |
| user_id       | Identifies the user creating the gene set                                                        |
| gene_set_uuid | UUID of gene set to be updated                                                                   |
| body          | JSON formatted dict containing a gene set (and accompanying genes) to be updated                 |
|               | {                                                                                                |
|               | "consensus_counter": "[Required] 0001",                                                          |
|               | "comments": "[OPTIONAL]Update to comments describing gene set, if null no changes will be made", |  |
|               | "genes_to_add":                                                                                  |
|               | [{                                                                                               |
|               | "gene name": "[Required] name of gene",                                                          |
|               | "comments": "[OPTIONAL] comments on why a gene was included in this gene set"                    |
|               | }],                                                                                              |
|               | "genes_to_delete": ["gene_names"],                                                               |
|               | "genes_to_change":                                                                               |
|               | [{                                                                                               |
|               | "gene name": "[Required] name of gene",                                                          |
|               | "comments": "updated comment"                                                                    |
|               | }],                                                                                              |
|               | }                                                                                                |

**Response:**
| Code | Description |
| ---- | --------------------------------------- |
| 200 | The gene sets were successfully updated |

| Key                | Description                                                                            |
| ------------------ | -------------------------------------------------------------------------------------- |
| consensus_counters | {"gene_set_uuid": "updated_counter"}                                                   |
| errors             | If an error occurred, a message specifying the error and the gene set it was linked to |
|                    | is returned to the client here                                                         |

**Error Responses:**

| Code | Description                             |
| ---- | --------------------------------------- |
| 400  | If the parameters supplied are invalid. |

| Code | Description                                                  |
| ---- | ------------------------------------------------------------ |
| 403  | User id invalid, or user does not have access to the dataset |

| Code | Description                                                   |
| ---- | ------------------------------------------------------------- |
| 500  | Consensus counters do not match, the error response will      |
|      | include the name and uuid of the gene set that is out of sync |

#### DELETE dataset_name/api/v0.1/genesets/{gene_set_uuid}

**Request:**

| Parameter     | Description                                         |
| ------------- | --------------------------------------------------- |
| dataset_name  | Identifies the dataset the gene sets are linked to. |
| user_id       | Identifies the user creating the gene set           |
| gene_set_uuid | UUID of gene set to be deleted                      |

**Response:**
| Code | Description |
| ---- | --------------------------------------- |
| 204 | The gene set was successfully deleted |

**Error Responses:**

| Code | Description                             |
| ---- | --------------------------------------- |
| 400  | If the parameters supplied are invalid. |

| Code | Description                                                  |
| ---- | ------------------------------------------------------------ |
| 403  | User id invalid, or user does not have access to the dataset |

| Code | Description                       |
| ---- | --------------------------------- |
| 404  | No gene set exists with that uuid |

## Alternatives

Because of the redundancy of some pieces of data in the csv (gene set name, cellset identifiers in the comments) we
considered using a json format instead, however discussions with scientists (our main users) made it clear that they
felt more comfortable editing and working with a csv file.

## References
