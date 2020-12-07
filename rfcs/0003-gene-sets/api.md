# Explorer User-Generated Gene Sets

## API Format

**Authors:** [Madison Dunitz](mailto:madison.dunitz@chanzuckerberg.com)

**Approvers:** [Arathi Mani](mailto:arathi.mani@chanzuckerberg.com), [Signe Chambers](mailto:schambers@chanzuckerberg.com), [Friendly Tech Person2](mailto:colin.megill@chanzuckerberg.com), [Girish Patangay](mailto:girish.patangay@chanzuckerberg.com), [Sara Gerber](mailto:sara.gerber@chanzuckerberg.com)

## tl;dr

This RFC contains the details about the API for the gene set feature. It will support the export of differential
expression results and allow a user to upload their own gene sets for analysis.

## Glossery

- Geneset - List of genes, also referred to as var_sets throughout this document
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

### Users can upload their own gene sets and see them in the right side bar with the full range of current cellxgene functionalities

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

## Detailed Design | Architecture | Implementation

Because gene sets are closely tied to differential expression, some aspects of the differential expression design plans will also be detailed below.

### User uploaded gene sets

#### 1. User creates csv containing a list of genes (and optional comments about why each gene was included)

CSV must follow format described in the Data Model section

#### 2. User uploads file to hosted cellxgene client

The user will typically upload a file from their local machine but we should consider supporting uploads from s3 in future iterations.

The user must be logged in to upload a gene set.

Users will upload the file by passing its path to the client.
The client will validate the format of the csv and check that the genes listed in the csv exist in the dataset

- if the format is invalid the client will raise a descriptive error message to the user
- if the file is valid, the client will display the genes in the right side bar
- if the file is valid, but it references genes that are not in the dataset, the client will display the genes that do
  exist in the right side bar and alert the user that their var set contained genes that are not a part of the dataset and they have
  been removed from the var set that will be stored/displayed.

If the user is logged in the frontend will send a POST request to {dataset_name}/api/v0.01/var_sets/ (see API
description below) with a json object containing the var sets and genes as the body of the request (see API
description below for more detail about the request body).

The backend will parse the json and create a var set (or sets), linking it to the user via the UserId field.
Genes listed in the var set will be created in the gene table and linked to the gene set via the VarSet field. A new
row will be created in the gene table for each gene included in a var set.

- If the gene set is successfully stored the backend will respond with a 201 and the uuid and consensus counter for that
  gene set (see keeping consensus below).

- If it is not successfully saved the backend will respond with a 400 and the names of the var sets that were not created.

No action will be taken by the client if the var set is successfully saved

If the backend returns a 400 the client will try again and then alert the user that their var set has not been saved.

### Export Differential Expression Output

#### Current Functionality

The user selects a set of cells via the lasso and then clicks the 1:0 cells button shown in Figure 2

The user selects a set of cells via the lasso and then clicks the 2:0 cells button shown in Figure 2

Once two sets of cells have been selected the button to the right of 2:0 cells (diffexp button) becomes available as shown in Figure 3. Figure 3 also shows that the cell selection button is updated to show the number of cells currently selected.

When the user selects the diffexp button, the client sends a request to the server which calculates the difference in
expression levels for all of genes between the two sets of cells. It then returns the 10 most differentially expressed
genes to the client which displays that information in the right side bar along with a histogram of their expression levels.

This information does not currently persist between sessions.

![Cellxgene Data Schema](imgs/diff_expression_button_pre.png)

**Figure 2** Cellxgene Diff Expression button pre cell selection

![Cellxgene Data Schema](imgs/diff_expression_button_post.png)

**Figure 3** Cellxgene Diff Expression button post cell selection

#### Additional Functionality

The gene sets created by a user will persist between sessions. When an authenticated user navigates to a particular dataset
the client will send a GET request to {dataset_name}/api/v0.01/var_sets retrieving all of the user's gene sets for
that dataset.

In the future we will limit the user's ability to create anonymous groups of cells. They will need to use the annotation
feature to create a category and label the cell group in order to select it for differential expression analysis.

The user will be able to export a CSV file containing one or all of the gene sets they created by selecting cell sets and
calculating the 10 most differentially expressed genes along with their logfold and p-values between those sets.

The comments section of the csv file will specify the two sets of cells being compared to create that gene set.

The name of the exported file will be based on the user name and a timestamp rounded to the nearest second.

### Keeping Consensus

This feature will require processing multiple updates to the same gene set, due to our hosting architecture this creates
the potential for race conditions and concerns about keeping the data in sync between the database, server and client.  
To ensure data consistency, when the user is interacting with the data, the client is always the source of truth.
If what is stored in the database doesnt match what the user is seeing, the data in the database will be discarded and
recreated based on the state of the client.

The client will ensure the data is kept in sync through a consensus token or counter. Everytime a var set is created
in the database it will have a consensus_counter field set to 0. This will be sent back to the client along with the
gene set name and uuid. When the client updates a gene set (via a PUT request) it will send this counter incremented by 1
back to the server as part of the request body. The server will confirm that the counter sent from the client is one
more than the counter stored in the database for that gene set.

- If the values is correct, the server will update the gene set as specified in the PUT
  request and update the counter with the value sent by the client. The response to the client will include the updated
  counter value to be used in future PUT requests for that gene set.
- If the counter stored sent by the client is equal to or less than the counter in the database the request will be ignored
  (under the assumption that it has already been processed)
- If the counter sent by the client is too high (more than 1 more than the counter stored by the client) the server will immediately send
  an error message to the client, specifying the name and uuid of the gene set that is out of sync as well as the value
  of the counter currently associated the gene set in the database.

  - The client can then send the missing updates (in order) followed by the failed update, OR
  - The client can send over the entire gene set including all linked genes as a POST request. Because the gene
    set name is already taken for that user/dataset, the server will delete the currently stored gene set and any
    associated genes and recreate it based on the information included in the POST request. This will generate a new UUID
    for the gene set, and reset the consensus counter.

  In the case that the server times out or does not respond to a PUT request from the client, the client can retry or
  just send a POST request to recreate the gene set.
  Because recreating the gene set assigns it a new UUID, any pending or high latency PUT requests sent prior to the update
  be ignored because they reference a UUID that no longer exists.

In the desktop version of the application the gene sets (and genes) will be stored locally as a csv under the name of
the dataset and the user session. On updates the application will create and store a new csv, maintaining the last
10 updates (this will match the current functionality for annotations in the desktop environment)

### Data Lifecycle

#### Creation

Gene sets are created via a POST request to `/dataset_name/api/v0.1/var_sets/`. This request is sent

- after a user uploads a csv continaing one (or more) gene sets.
- based on the results of comparing differential expression of two groups of cells.
- after the user creates a new gene set via the application GUI

Genes are created as a part of the gene set creation step. They can also be added to an existing gene set via a PUT
request to `/dataset_name/api/v0.1/var_sets/{var_set_uuid}`

#### Updates

Gene sets are updated via a PUT request to `/dataset_name/api/v0.1/var_sets/{var_set_uuid}`.

Genes can also be updated via this mechanism

#### Deletion

Gene sets are deleted via a DELETE request to `/dataset_name/api/v0.1/var_sets/{var_set_uuid}`. All genes attached to
the gene set will be deleted as well.

Genes can be removed from a gene set by including their name in the `genes_to_delete` list that is part of the
request body for PUT requests to `/dataset_name/api/v0.1/var_sets/{var_set_uuid}`

### Data model

Fully supporting persistence of differential expression results is out of the scope of this RFC. But the data model described below has been designed to allow for differential expression persistence in the future.

### Relational Database

To support gene sets two tables will be added to the cellxgene relational database

- Gene
  - UUID (generated)
  - GeneName
  - Comments
  - VarSet
  - created_at
  - updated_at
- VarSet
  - UUID (generated)
  - ConsensusCounter
  - VarSet Name (required)
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

### APIs

All APIs assume user identification is available with the request and the dataset being worked on is specified in the url
Future API endpoints for consideration

- LIST dataset_name/api/v0.1/var_sets/ - list all gene sets for a dataset (just names)
- GET dataset_name/api/v0.1/var_sets/{var_set_uuid} - get all information for one particular gene set

#### Get dataset_name/api/v0.1/var_sets/

Return all of the var sets (and accompanying genes) associated with a given user/dataset.

**Request:**
| Parameter | Description |
| ------------ | --------------------------------------------------------------------------------------------- |
| dataset_name | Identifies the dataset the var sets are linked to. |
| user_id | Identifies the user linked to the var set. |

**Response:**

| Code | Description                              |
| ---- | ---------------------------------------- |
| 200  | The gene sets were successfully returned |

| Key  | Description                                                                        |
| ---- | ---------------------------------------------------------------------------------- |
| data | List of var set dicts containing their identifying information and a list of genes |

```json
{
  "var_sets": [
    {
      "var_set_name": "gene set name",
      "var_set_uuid": "uuid",
      "consensus_counter": 1,
      "comments": "Comments describing gene set",
      "genes": [
        {
          "gene name": "name of gene",
          "comments": "comments on why a gene was included in this gene set"
        }
      ]
    }
  ]
}
```

**Error Responses:**

| Code | Description                             |
| ---- | --------------------------------------- |
| 400  | If the parameters supplied are invalid. |

| Code | Description                                                  |
| ---- | ------------------------------------------------------------ |
| 403  | User id invalid, or user does not have access to the dataset |

#### POST dataset_name/api/v0.1/var_sets/

Any gene sets created or uploaded (via the csv format described above) will be sent via json to the backend for long term
storage/persistence.
Gene sets and gene names must be unique for a user/dataset - if an already used gene set name is submitted, the backend
will assume it is due to a consensus issue and delete the currently stored gene set (and any attached genes) and
recreate it based on the information in the request

**Request:**
| Parameter | Description |
| -------------- | --------------------------------------------------------------------------------------------- |
| dataset_name | Identifies the dataset the gene sets are linked to. |
| user_id | Identifies the user creating the gene set |  
| body | JSON formatted dict containing a list of gene sets (and accompanying genes) to be created |

```json
{
  "var_sets": [
    {
      "var_set_name": "[Required] var set name",
      "comments": "[OPTIONAL]Comments describing var set",
      "genes": [
        {
          "gene name": "[Required] name of gene",
          "comments": "[OPTIONAL] comments on why a gene was included in this gene set"
        }
      ]
    }
  ]
}
```

**Response:**
| Code | Description |
| ---- | --------------------------------------- |
| 201 | The var set(s) were successfully saved |

| Key     | Description                                                                              |
| ------- | ---------------------------------------------------------------------------------------- |
| var_set | Dict mapping the var set name to its assigned uuid and its consensus counter             |
| errors  | In the case where some gene sets were successfully stored and others will not, a message |
|         | specifying the error and the var set(s) it was linked to is returned to the client here. |

```json
{ "name_of_var_set": { "uuid": "var_set_uuid", "consensus_counter": 0 } }
```

**Error Responses:**

| Code | Description                             |
| ---- | --------------------------------------- |
| 400  | If the parameters supplied are invalid. |

| Code | Description                                                  |
| ---- | ------------------------------------------------------------ |
| 403  | User id invalid, or user does not have access to the dataset |

#### PUT dataset_name/api/v0.1/var_sets/{var_set_uuid}

Any changes to a gene set (the gene set itself or genes within that gene set) will be sent via json to the backend.
This includes the creation/deletion of genes within a gene set
Updates to comments will overwrite the currently stored comments
**Request:**

| Parameter    | Description                                                                                 |
| ------------ | ------------------------------------------------------------------------------------------- |
| dataset_name | Identifies the dataset the gene sets are linked to.                                         |
| user_id      | Identifies the user creating the gene set                                                   |
| var_set_uuid | UUID of var set to be updated                                                               |
| body         | JSON formatted dict containing a var set (and accompanying genes) to be updated-- see below |

```json
{
  "consensus_counter": 1,
  "comments": "Update to comments describing var set, if null no changes will be made",
  "genes_to_add": [
    {
      "gene name": "name of gene",
      "comments": "Comments on why a gene was included in this gene set"
    }
  ],
  "genes_to_delete": ["gene_names"],
  "genes_to_change": [
    {
      "gene name": "Name of gene",
      "comments": "updated comment"
    }
  ]
}
```

**Response:**
| Code | Description |
| ---- | --------------------------------------- |
| 200 | The var sets were successfully updated |

| Key                | Description                                   |
| ------------------ | --------------------------------------------- |
| consensus_counters | {"var_set_uuid": "current_consensus_counter"} |

**Error Responses:**

| Code | Description                             |
| ---- | --------------------------------------- |
| 400  | If the parameters supplied are invalid. |

| Code | Description                                                  |
| ---- | ------------------------------------------------------------ |
| 403  | User id invalid, or user does not have access to the dataset |

| Code | Description                                                                                   |
| ---- | --------------------------------------------------------------------------------------------- |
| 500  | Consensus counters do not match, the error response will include value of the consensus       |
|      | counter currently stored in the db, and the name and uuid of the gene set that is out of sync |

#### DELETE dataset_name/api/v0.1/var_sets/{var_set_uuid}

**Request:**

| Parameter    | Description                                         |
| ------------ | --------------------------------------------------- |
| dataset_name | Identifies the dataset the gene sets are linked to. |
| user_id      | Identifies the user who owns the var set            |
| var_set_uuid | UUID of var set to be deleted                       |

**Response:**
| Code | Description |
| ---- | --------------------------------------- |
| 204 | The var set was successfully deleted |

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

## Questions

- [ ] Best way to handle var sets linked to the dataset (available to all users) with no owner. `is_owned_by_dataset` field on var_set table?
- [ ] Request/Response size limits? What does that translate to in terms of # gene sets or number of genes per gene set?
