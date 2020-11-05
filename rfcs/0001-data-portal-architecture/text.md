# Corpora High Level Architecture

Authors: [arathi.mani@chanzuckerberg.com](mailto:arathi.mani@chanzuckerberg.com), [bruce@chanzuckerberg.com](mailto:bruce@chanzuckerberg.com)

Document Status: _Approved_

Date Last Modified: 2020-05-29

_tl;dr: This document describes the high level architecture of the Corpora backend, specifically focusing on the necessary backend that needs to be built in order to get to the MVP._

**The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED" "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14, RFC2119, and RFC8174 when, and only when, they appear in all capitals, as shown here.**

## Background

For [the first milestone of Corpora](https://docs.google.com/document/d/13KbnfQVruyEp5JbeTbhelYejroqBHn2jtW0YbRnAx0I/edit#heading=h.vcwscwt15zqy), we are aiming to fulfill two user experiences:

1. A prospective publisher understands what metadata is required when submitting a dataset to Corpora. Manual interactions between the publisher and curator continue.
2. The ‘find data’ portal on `corpora.cziscience.com` portal is “live” on the internet with migrated COVID-19 datasets from `cellxgene.cziscience.com`. Consumers can view the Corpora metadata for each dataset and launch cellxgene.

During this first milestone, we will aim to also establish the groundwork for the high level architecture that will enable us to painlessly build for future milestones enable such interactions with the portal as the automated ability to upload a new collection, the ability to update a collection, the ability to delete a collection, and more.

With both the near term goals as well as future direction in mind, the following document lays the foundation for the data portal architecture that will be the first point of interaction between Corpora and a Data Submitter and will enable the cellxgene portal to harness the collection of data for valuable downstream analysis.

## Product Decisions

While probably not comprehensive, the following product decisions were made in parallel to the writing of this document - these decisions are related to the criteria for submitting a dataset which have been further discussed [here](https://docs.google.com/document/d/1_I2dnYuCTvO2yABR0FnLRnI8RnMhiASC5Ny4ZwDjsCI/edit#heading=h.jdvqjtitikjv):

- File formats available for download: {`.loom`, `.rds` (Seurat object), `.h5ad`}
- Files available for download: the original submitted matrix (in any format) and the [Remix matrix](https://docs.google.com/document/d/1_I2dnYuCTvO2yABR0FnLRnI8RnMhiASC5Ny4ZwDjsCI/edit#heading=h.jdvqjtitikjv) (in any format)
- All file formats will be generated via the H5AD form of either the submitted matrix or the remix matrix (See Figure 2 for a clearer picture).

## Detailed Design

### Design Principles

At times, we have had to make tradeoffs between different options and we’ve converged upon a few “rules-of-thumb” that has helped break the ties. These design principles should be propagated to further refinement of component-design as we progress as well:

1. Choose simplicity when possible.
2. Design for the narrow use case as long as it doesn’t block us from expansion for the general use case.

### High-level Architecture

In order to keep things relatively simple and based off of previous design work and iterations of Data Portal-like work, the following is a high-level overview of the Corpora architecture.

While of course there will be further architectural pieces built out over time, this provides an initial draft that provides an overview of the underlying mechanisms that will occur in order to process datasets and explore datasets with some high level details such as an API, Data Model, and Processing Pipeline functionality fleshed out.

Notably missing in this first draft is the authorization/authentication piece which will be a shared service across both the Data Portal and cellxgene.

![alt_text](imgs/high_level_abstract_architecture.png)

<p style="text-align: center;"><b>Figure 1</b>: High-level architecture of the data portal for the first draft (note the human manually adding new datasets as a user of the backend data portal functionality which will eventually be non-existent).</p>

#### Collection Processing Stages/Event Definitions

The major part of the overall system is the middle layer of centralized data processing. These are the steps that occur once a submitter is ready to submit a collection for either viewing privately or publicly and propagates the collection and associated matrix files to the Corpora downstream services, i.e. the Data Portal and cellxgene.

![alt_text](imgs/flow_diagram_of_downloadable_artifacts.png)

<p style="text-align: center;"><b>Figure 2</b>: Flow diagram showing the ancestry of downloadable artifacts and how they were generated.</p>

The following steps/stages are available for subscription in the batch data processing pipeline, and are named by the triggers which initiate them:

- **MatrixValidate** - initial input validation, intended to be fast (interactive speed), trivial rejection of common errors. Passing this stage without error does not guarantee that matrix publication will succeed.
- **MatrixCreate** - create the remix data matrix, from which all other work proceeds.
- **AssetsCreate** - transform the collection and matrices into other formats and/or representations needed for deployment.
- **AssetsDeploy** - deploy the matrix as necessary (e.g., put a copy into the cellxgene deployment bucket)
- **AssetsCleanup** - cleanup/delete any deployments of the collection/matrix.

Each of these steps **<span style="text-decoration:underline;">MUST be idempotent</span>**, and MUST operate without side-effects other than those noted.

We considered merging the AssetsCreate and AssetsDeploy steps together; there were two primary reasons why it was rejected:

1. The transformation functions are the most likely to take the longest time as well as fail so we wanted to decouple failure of the transformation from deployment failure. This will also enable easier debugging in the long run.
2. Decoupling the creation step allows us to generalize the practice of generating derived products based on the user-inputted data. In the future if we need to create additional derived assets (i.e. a report of some sort), then we’d want to decouple the event of creating the new products from its deployment which may be shared.

![alt_text](imgs/flow_of_data_through_pipeline.png)

<p style="text-align: center;"><b>Figure 3:</b> Flow of data through events that occur in the data processing pipeline after a “publish” API call is made. Original Lucidchart is <a href=https://app.lucidchart.com/invitations/accept/4bd03ae5-8ae8-402d-97d9-2db0da8fbcca>here</a>.</p>

##### MatrixValidate

**Purpose**: The MatrixValidate step performs validation steps that take longer than interactive time but does not involve computation or transformation of the matrix itself. Any validation that occurs at interactive speed will be handled at the application layer while interacting with the UI and longer deep validation that usually is surfaced via the transformation/computation step will be surfaced via execution of the “Create” steps below. Example validations that occur at this step include rejecting a dataset due to PII and adherence of the matrix to the Corpora schema (see Validation section below).

- **Trigger**: availability of submitter-provided collection and matrix; API driven on-demand validation.
- **Constraints**: Must accept [any submission dataset format](https://docs.google.com/document/d/1_I2dnYuCTvO2yABR0FnLRnI8RnMhiASC5Ny4ZwDjsCI/edit#heading=h.jdvqjtitikjv).
- **Invocation parameters**:

  ```js
  {
    task_id, // unique ID of this task, suitable for logs
    collection_uuid,
    dataset_uuid,
    collection: {}, // inline definition of all collection fields
    submission_uri, // data locator for the dataset, e.g. s3:...
    submission_type, // encoding type of the dataset, e.g. H5AD
  }
  ```

- **Side effects/results**: this processing step must return a success/failure indication, along with additional information about any erroneous results. The validation status will also be updated in the central database signalling successful validation.

  ```js
  {
    task_id,
    success: true/false,
    message, // human readable information on failure to validate
  }
  ```

- **Notes**: this event may occur repeatedly for a collection/dataset (e.g. upon each update), and both collection and dataset may be partially specified (this means that optional fields may not be present*)*. Collections containing multiple datasets will have the event dispatched at least once per (collection, dataset) pair.

##### MatrixCreate

**Purpose**: create the remix matrix, e.g. populate the matrix with any mandatory fields, normalize metadata, etc. The result of this processing stage is a single dataset from which all other alternative encodings of the dataset will be generated.

- **Trigger**: MatrixValidation event completes and sets a VALID state on a collection; collection & dataset is ready for conversion and deployment.
- **Constraints**: must accept any submission dataset format. There must be <span style="text-decoration:underline;">one and only one</span> registered handler for this event/invocation, per (collection,dataset) pair.
- **Invocation parameters**:

  ```js
  {
    task_id,
    collection_uuid,
    dataset_uuid,
    collection: {}, // inline definition of all collection fields
    submission_uri, // data locator for the submission, eg, s3:...
    submission_type, // encoding type of the submission, eg, H5AD
    result_uri, // result dataset name, eg, s3:.../foo.loom
  }
  ```

- **Side effects/Result**: the resulting dataset must be stored in the location specified by "result_uri", and must be of the type specified by "result_type"

  ```js
  {
      task_id,
      success: true/false,
      message, // human readable information on failure to create
  }

  ```

- **Notes**: n/a

##### AssetsCreate

**Purpose**: create alternative \_temporary \_assets from the collection and remix matrix. For example, this might create all of the file formats required to support download, or other intermediate formats required for deployment. Will be invoked once per (collection, dataset). Assets will be removed/destroyed after deployment. This step can potentially take up to hours on large datasets.

- **Trigger**: successful completion of MatrixCreation step
- **Constraints**: must accept whatever formats the MatrixCreation step generates.
- **Invocation parameters**:

  ```js
  {
      task_id,
      collection_uuid,
      dataset_uuid,
      collection: {}, // inline definition of all collection fields
      src_dataset_uri, // the dataset location
      src_dataset_type, // the dataset type, eg, "loom"
      dest_uri, // destination container for assets, eg, s3:...
  }
  ```

- **Side effects/results**: any intermediate asset output of any conversion must be stored into the "dest_uri". Each result must be returned:

  ```js
  {
    task_id,
    success: true/false,
    message, // human readable information on failure to transform
    assets: {
      converted_cxg: foo.cxg,
      converted_h5ad: bar.h5ad,
      converted_loom: baz.loom,
      converted_seurat: qux.rds,
    } // one or more resulting assets residing in the given dest_uri
  }
  ```

- **Notes**: the output of this stage SHOULD be deleted after successful deployment - i.e. assets are temporary in nature. However they also MAY be cached, as this operation MUST be idempotent.

##### AssetsDeploy

**Purpose**: deploy the collection, with any/all assets, into application-specific containers/systems. It is expected that all individual downstream portals will have their own deployment handler. This step should be used for database loading, asset staging, etc. All temporary assets will be deleted. This step will usually take on the order of minutes to complete.

- **Trigger**: system has decided that a collection is ready to be visible to someone (see note).
- **Constraints**: successful completion of MatrixValidate, MatrixCreate & AssetsCreate steps.
- **Invocation parameters**:

  ```js
    {
        task_id,
        collection_uuid,
        collection: {}, // inline definition of all collection fields
        datasets: [ // list of all datasets in this collection
            uuid,
            type,
            uri
        ],
        assets_uri, // location containing output of Transform step \
    }
  ```

- **Side effects/results**:

  ```js
  {
      task_id,
      success: true/false,
      message, // human readable information on failure to deploy

  }
  ```

- **Notes:** deployment does not imply \_public visibility \_of the collection or dataset -- that state is managed live, in the main database. It is also expected that all dataset-transformation-related error handling will have occurred prior to this step, and this step will only concern itself with deployment.

##### AssetsCleanup

**Purpose**: delete a collection and all included datasets, OR a set of datasets from a deployment.

- **Trigger**: user or system need to delete
- **Constraints:** none - could occur even if other steps have not occured, e.g. as a conservative error handling step.
- **Invocation parameters**:

  ```js
      {
          task_id,
          collection_uuid,
          dataset_uuid: [], // list of datasets
          delete_entire_collection: true/false [default=False],
      }
  ```

- **Side effects/results**: any dataset listed will be undeployed/deleted. If "delete_entire_collection" is true, all other collection-related data must be deleted.

```js
  {
      task_id,
      success: true/false
   }
```

- **Notes**: if `delete_entire_collection` is `false`, only the subset of datasets specified need be deleted. Other collection information and datasets should be left as-is.

### Other Services

Besides the data processing pipeline, there are a number of other services that will need to be implemented:

- Pipeline monitoring [NOT PART OF MVP]
- This service should allow the UI to display what stage of processing the collection has undergone.
- Authentication service [Milestone 2]
- Trent to design
- Corpora will also maintain a cellxgene URL “directory.” Corpora backend will be responsible for generating unique URLs that map to a deployment of cellxgene for a dataset and also maintain aliases that will be available for user consumption based on the visibility of the collection (i.e. private, shared, or public).

![alt_text](imgs/high_level_aws_architecture.png)

<p style="text-align: center;"><b>Figure 4</b>: High level view of the overall architecture of Corpora. The big red boxes, Authentication and Data Processing will be designed by Trent and Calvin respectively. Some lines have been omitted for cleanliness and simplicity and primarily shows the forward flow of data through the system. Authentication is heavily simplified as in practice, it will permeate the entire system including deployment. Original link is <a href=https://app.lucidchart.com/invitations/accept/5a20e8c6-ac40-49ee-bf2f-6f73216de6ff>here</a>.</p>

### Technologies [BRAINSTORM]

- A PostgreSQL instance of Amazon RDS will host all the metadata and meta-metadata about collections, submissions, datasets, and more. More details are below in the Data Model section of this document.
- Postgres allows simplification of access control by using row level access controls that are built-in.
- S3 will host all of the data associated with a collection including matrix files, completed Terms and Conditions forms, and any other associated files when necessary.
- APIs will be published on Amazon API Gateway and run on Amazon Lambda, making use of Chalice. The APIs will be defined using OpenAPI v3.0.
- Deployment through Elastic Beanstalk to match cxg?

### Data Flow

#### Inputting a New Collection

Following the Corpora Contributor User/System flow outlined [here](https://drive.google.com/drive/folders/1tjCt-lXAceuRMUZRzCcMeCO2NjVfI6ac), the following actions will take place in the software system.

1. Missing - Authentication for a user to be allowed to submit a new collection.
2. User chooses to add new collection
3. API call **`POST /v1/submissions`** with a `needs_attestation` parameter `True/False` is called and a `collection_uuid` is a generated collection UUID. This action will create a new row in the Submission and Collection tables that are empty and will create a new folder in the S3 bucket with the given `collection_uuid`. If `needs_attestation` is specified as `true`, this will flip the `needs_attestation` field in the newly created rows to be true so that the collection cannot be submitted without the appropriate legal files uploaded as well.
4. User uploads datasets that are part of the collection and optionally chooses to fill out a form with required (and optional) metadata. If the form is not filled out, the collection-level metadata will be inferred from the first uploaded dataset.
5. Uploading matrix files
   1. API call **`PUT /v1/submissions/{collection_uuid}/{filename}/{data_type}/data`** uploads each of the files into the S3 bucket with the folder with the given collection UUID. This will be called once per dataset. See the API Design below for more details.
6. Uploading Terms and Conditions file
   2. API call **`PUT /v1/submissions/{collection_uuid}/{filename}/{data_type}/legal`** uploads the file into the S3 bucket with the folder with the given collection UUID and updates the record of the submission with the S3 URI in the database. See the API Design below for more details.
7. No action is taken on the collection’s metadata until the submit button is clicked.

#### Saving a New Collection

Saving a new collection represents one of three actions that can take place after “Inputting a New Collection” is completed. Saving a collection does not run any pipeline processing and simply saves any inputted collection-level metadata in the Submission tables. A simple propagation step will be run if necessary in order to extract collection-level metadata from the dataset(s) and enter it into the RDS. No deep validation is run (a small sanity check will be run) and the collection remains private.

1. User clicks “Save.”
1. API call **`POST /v1/submissions/{collection_uuid}/save`**.

#### Publishing a Collection Privately

Publishing a collection privately runs the full data processing pipeline on the given collection details and datasets and delivers a URL where the submitted may view their generated collection page as well as their dataset(s) in cellxgene. This URL, under the hood, will point to a permanent URL but will be unindexed and obfuscated (though sharable). Publishing private is the second of three possible steps that can take place after “Inputting a New Collection.” This step is also repeatable even in the MVP to allow incremental edits.

1. User clicks a button
1. API call **`POST /v1/submissions/{collection_uuid}/publish/private`**
1. Collection information is pulled from the form and entered into the database (same execution as the save API call).
1. At this point, the Data Processing Pipeline will be launched with parameters set such that the resulting assets/deployments are only viewable by people who own the secret. The secret will be stored in the database as well.

#### Publishing a Collection Publicly

Publishing a collection publicly runs the full data processing pipeline (in a future iteration, we will check for deltas against a cache so that the processing pipeline as an optimization), and then stores public facing links that are indexed in the data portal for public consumption. Under the hood, we will maintain a directory of private URLs and public URLs where the public facing URLs are the only ones that are indexed by the data portal for larger visibility (i.e. cellxgene deployments will be launchable from the data portal).

1. User clicks a button
1. API call **`POST /v1/submissions/{collection_uuid}/publish/public`**
1. Collection information is pulled from the form and entered into the database (same execution as the save API call).
1. At this point, the Data Processing Pipeline will be launched with parameters set such that the resulting assets/deployments are fully public.
1. Once completed, if there is an existing row in the Collection database with the same collection UUID as the one that is being published, in a single transaction, the existing row will be deleted and the new row's `status` attribute will be set to `Live`.

#### Deleting a Collection

For now, let's limit the deletion of collections to be an engineer/admin-only procedure. Can be done by calling **`DELETE /v1/submissions/{collection_uuid}/{filename}`**. Individual datasets can be deleted using the same command.

#### Microservice Deployment Replay

Deployment replays may occur if one of the downstream portals needs to make a sweeping change that requires re-deployment of existing collections and will likely occur more often, earlier in the Corpora collection. In this scenario, there will be no API-driven “publish” event that occurs because the user has already submitted. Instead a downstream service can use one of the two API calls with filters to “replay” a publish to re-deploy collections.

1. Use **`GET /v1/collections/list`** and/or **`GET /v1/collections/{collection_uuid}`**

### Data Model

#### ER Diagram

The below ER (entity relation) diagram represents the core entities that will be kept track of by the data portal backend and specifically is considerate of the necessary data in order to populate the frontend web browser and corresponding functionality. There are a few specific things to note here:

1. This data model does not denormalize the matrix that is associated with a data and associated metadata that is available there or in the finer-grained “Observation” and “Feature” entities. The requirements for populating the metadata are not engineering-driven, as in the metadata within the matrix itself is not used to populate the database backend.
2. Data model does not currently exhibit necessary information related to Authentication/Authorization. This will be filled in at a later point in time with the AuthN/AuthZ design.

![alt_text](imgs/er_diagram.png)

<p style="text-align: center;"><b>Figure 5:</b> ER diagram modeling the core entities of the Data Portal as well as the relationships between the entities. The original Lucidchart that should be edited is <a href=https://app.lucidchart.com/invitations/accept/3672a8a1-ebfb-4411-94c1-d8f3f7d9bd85>here</a>. The blue attributes are ones that are provided by the data contributor. The rest are either system related or calculated.</p>

#### UML of Database

Based on the ER diagram about and ensuring that the database design is in Boyce-Codd normal form, the following UML describes the RDS.

![alt_text](imgs/original_relational_diagram.png)

<p style="text-align: center;"><b>Figure 6</b>: Core Corpora database object relational diagram. The original Lucidchart that will be edited and kept up is <a href=https://app.lucidchart.com/invitations/accept/039b8ec2-81a2-4916-906f-e5a71851369a>here</a>.</p>

#### Submission vs. Collection Entities

One of the more confusing parts of the above data model is distinguishing between the concept of a submission and a collection. Using the Github metaphor is helpful:

- A **collection** is the master branch of a Github repository.
- A **submission** is a pull request on a particular repository (or **collection**).

For simplicity, we can say that for now, there can only be one open submission on a collection at any given time (boom, merge conflicts resolved!). There are a few characteristics that fall out quite nicely with this metaphor. A collection entity can be seen as the public facing immutable entity that has public facing cellxgene URL links and is searchable in the data portal. A submission allows a user to edit a collection, _even previously published ones_, and fiddle with their live changes until satisfaction at which point the publication _replaces_ the existing collection (under the hood, it might not be a full delete->recreate, but we’re talking conceptually here) thus enforcing immutability.

### API Design

`GET /v1/submission`

**Description**: Lists all submissions by their UUIDs that currently exist in Corpora. If a parameter is specified as a filter, then only submissions that meet the given criteria will be returned.

**Parameters**:

1. User_ID [OPTIONAL]: an ID that represents the logged in user. Returned submissions will be only ones that the given user has created.

**Responses:**

1. 200 OK: `{request_id, collection_uuid, name, processing_state, validation_state, user_id}`
2. 400 Invalid parameter

`GET /v1/submission/{collection_uuid}`

**Description**: Returns all available metadata information about a collection submission, including URIs of datasets that are attached to the collection.

**Parameters**:

1. Collection UUID [REQUIRED]: the UUID of the collection on which information is being requested.

**Responses:**

1. 200 OK
1. Response payload will be a schema with all fields that are associated with a submission.
1. 401 Unauthorized user
1. 404 Submission not found

`PUT /v1/submission/{collection_uuid}/file`

**Description**: Adds a dataset file or a legal file to the collection’s S3 bucket. If a legal file is uploaded (verified by a standard name), then the requirement for attestation for the collection will be satisfied and accordingly noted in the submission entity. A quick validation will be performed to ensure that if the file is a data file, then the extension is one of the accepted types (i.e. `.loom`, `.h5ad,` or `.rds`). If a file already exists with the same name, it will replace it and increment the revision value of the dataset.

**Parameters:**

1. Collection UUID [REQUIRED]: the UUID of the collection to which the file needs to be added.
2. Filename [REQUIRED]: Request body parameter. The name of the file that is being uploaded. A duplicate filename, will result in the overwrite of the existing file.
3. Filetype [REQUIRED]: Request body parameter. the filetype will be one of `{legal or data}` which provides information on how to handle the file. If legal, then an update is made to the Submission entity signifying that the T&C form has been submitted.
4. Body {REQUIRED]: Request body parameter. The contents of the file to be added to the submission.

**Responses:**

1. 201 Created: `{status, s3_uri, dataset_uuid}`
2. 400 Invalid filename or filetype: `{status, message}`
3. 401 Unauthorized: `{status, message}`
4. 404 Submission not found: `{status, message}`

`DELETE /v1/submission/{collection_uuid}`

**Description**: Deletes the submission associated with the given collection UUID. This does not delete the collection if the collection associated with the submission has been previously publicly published.

**Parameters**:

1. Collection UUID [REQUIRED]: the UUID of the collection to be deleted.

**Responses:**

1. 202 Accepted
2. 401 Unauthorized user
3. 404 Collection not found

`DELETE /v1/submission/{collection_uuid}/dataset`

**Description**: Deletes a file from the collection’s S3 bucket if the file exists. If no such file exists, then a warning will be outputted.

**Parameters**:

1. Collection UUID [REQUIRED]: the UUID of the collection from which the file will be removed.
2. Dataset UUID [REQUIRED]: Request body parameter. The UUID of the dataset that is being removed.

**Responses:**

1. 202 Accepted
2. 401 Unauthorized user
3. 404 Submission or file not found

`POST /v1/submission`

**Description**: Opens a new submission with a unique UUID if a submission is not already open for a collection. If a collection UUID is not provided, then a collection UUID will be generated. Otherwise, the collection details based on the given collection UUID will be used to pre-populate the newly created submission entity. If no collection UUID is given, then this call will also create a new folder with the generated collection UUID in the data portal S3 bucket. On success, a message will be returned with the collection’s UUID.

**Parameters**:

1. Needs Attestation [REQUIRED]: determines whether this collection requires attestation in which case this collection cannot be submitted without a Terms and Conditions file also uploaded.
2. Collection UUID [OPTIONAL]: If the submissions is created in order to update an existing publicly published collection, then a Collection UUID must be provided in order to backfill the fresh submission entity with the existing collection details for modification.

**Responses:**

1. 201 Created: `{collection_uuid}`
2. 401 Unauthorized user

`POST /v1/submission/{collection_uuid}/validate`

**Description**: Validates the collection according to the guidelines below in the Validation section and outputs the result, specifying all errors. The result of the validation is also stored in the backend database.

**Parameters**:

1. Submission UUID [REQUIRED]: The submission UUID of the submission that is being validated.

**Responses:**

1. 200 OK: `{collection_uuid, result, message}`
2. 401 Unauthorized user
3. 404 Submission not found

`POST /v1/submission/{collection_uuid}/save`

**Description**: if needed, extracts the collection-level metadata from the datasets or uses the given collection-level metadata imputed via the body parameter and saves it to the database. The new collection metadata state will be returned in the response on success.

**Parameters**:

1. Body [OPTIONAL]: `{name, description, raw_data_link, protocol_link, other_information_link}` a schema that has fields for all available collection-level metadata.

**Responses:**

1. 200 OK: `{collection_uuid}`
2. 400 Unsupported user-supplied schema fields
3. 401 Unauthorized user
4. 404 Submission not found

`POST /v1/submission/{collection_uuid}/publish`

**Description**:

**Parameters**:

1. Body [REQUIRED]: a schema that contains a field dictating the visibility of the deployments that will either make the collection visible on the public sites or only viewable through [obfuscated URLs](https://www.w3.org/TR/capability-urls/). If the visibility is set to public, the successful execution of this operation will move the Submission to a CLOSED state.

**Responses:**

1. 202 OK: `{collection_uuid, visibility}`
2. 400 Invalid visibility value
3. 401 Unauthorized user
4. 404 Submission not found

---

`GET /v1/collections`

**Description**: this lists all collections and their UUIDs that currently exist in the data portal. If a parameter is specified as a filter, then only collections that meet the status criteria will be outputted.

**Parameters**:

1. User UUID [OPTIONAL]: the UUID of the user who submitted/created the collection.
2. From date [OPTIONAL]: the date after which collections should have been created.
3. To date [OPTIONAL]: the date before which collections should have been created.

**Responses:**

1. 200 OK
1. Response payload will be a schema with fields `request_id`,`collection_uuid`, `user_id`,`from_date`, and `to_date`.
1. 400 Invalid query parameter

`GET /v1/collections/{collection_uuid}`

**Description**: this will return all datasets attributes and associated attributes of a collection with the given UUID.

**Parameters**:

1. Collection UUID [REQUIRED]: the UUID of the collection.

**Responses:**

1. 200 OK
1. Response payload will be a schema with all available metadata about the collection.
1. 401 Unauthorized user
1. 404 Collection not found

`DELETE /v1/collections/{collection_uuid}`

**Description**: Deletes an entire collection from Corpora, including any generated artifacts/assets and deployments. If no such collection exists, then a warning will be outputted.

**Parameters**:

1. Collection UUID [REQUIRED]: the UUID of the collection to be deleted.

**Responses:**

1. 200 OK
2. 401 Unauthorized user
3. 404 Collection not found

---

`POST /v1/dataset/{dataset_uuid}/asset/{asset_uuid}`

**Description:** Request to download a file which on success will generate a pre-signed URL to download the dataset.

**Parameters:**

1. Dataset UUID [REQUIRED]: the UUID of the dataset containing the asset file to be downloaded.
2. Asset UUID [REQUIRED]: the UUID of the asset file to be downloaded.

**Responses**:

1. 200 OK: `{dataset_uuid, presigned_url, file_name, file_size}`
2. 401 Failed to authenticate
3. 403 Unauthorized user
4. 404 File not found

## Validation

### Checksums

Do checksumming; SHA-256? Could do the 4 checksums as done in Upload Service.

### Data Files Validation

#### Quick Validation/Sanity Checks

Quick validation is done in interactive time so that the submitter can quickly catch easy mistakes. Successfully sanity checking does not update any database records to show successful validation. That can only be completed after the long-form validation.

- Matrix files is one of the accepted types: `{.loom or .h5ad or Seurat object}`
- If the collection needs attestation, make sure an attestation file has been submitted.

#### Lambda Validation

- Verify that the inferred dataset file type based on dataset file extensions actually matches the contents.
- i.e. .rds contains a Seurat object, .loom contains a Loom file, etc.
- Matrix file does not contain PII, which are listed [here](https://docs.google.com/document/d/1nlUuRiQ8Awo_QsiVFi8OWdhGONjeaXE32z5GblVCfcI/edit)
- Check all schema requirements as detailed [here](https://docs.google.com/document/d/1_I2dnYuCTvO2yABR0FnLRnI8RnMhiASC5Ny4ZwDjsCI/edit#heading=h.i5w3qbht4wai)

## Future Topics/Open Questions

There are a number of topics that are briefly mentioned in this document which are not delved into more deeply as they aren’t necessarily required for the bare minimum scaffolding to get the Corpora collection going (which was the intent of this document). The list of topics, to be discussed later, is listed below:

- Pipeline monitoring
- This service should allow the UI to display what stage of processing the collection has undergone and also provide a notification to the user when the pipeline has completed processing so that they can check on their results.
- Authentication service [Milestone 2]
- Under the assumption that Auth is a shared service across all Corpora portals, we need to determine what profile data needs to be stored (big red box in the Data Model UML diagram) as well as generally how AuthN/AuthZ will be shared.

## Approvers

Deadline for Review: 2020-05-29

Stakeholders, please write LGTM + the date in the box next to your name after reviewing the document like this: **LGTM 2020-MM-DD.**

<table>
<tr>
 <td>Bruce Martin
 </td>
 <td><strong>Initial review done 2020-05-26, left questions inline</strong>
 </td>
 <td>Lia Prins
 </td>
 <td><strong>LGTM 2020-05-26</strong>
 </td>
</tr>
<tr>
 <td>Marcus Kinsella
 </td>
 <td><strong>Not Reviewed</strong>
 </td>
 <td>Brian Raymor
 </td>
 <td><strong>Not Reviewed</strong>
 </td>
</tr>
</table>

## Reviewers

Deadline for Review: 2020-05-29

Stakeholders, please write LGTM + the date in the box next to your name after reviewing the document like this: **LGTM 2020-MM-DD.**

<table>
<tr>
 <td>Madison Dunitz
 </td>
 <td><strong>LGTM 2020-05-28</strong>
 </td>
 <td>Trent Smith
 </td>
 <td><strong>2020-05-28</strong>
 </td>
</tr>
<tr>
 <td>Brian McCandless
 </td>
 <td><strong>2020-05-28</strong>
 </td>
 <td>Calvin Nhieu
 </td>
 <td><strong>LGTM 2020-05-26</strong>
 </td>
</tr>
</table>

## References

\[0] [Single Cell Short-term Strategy](https://docs.google.com/document/d/13KbnfQVruyEp5JbeTbhelYejroqBHn2jtW0YbRnAx0I/edit#heading=h.8oshqs8we1iz)

\[1] [Cellxgene Dataset Flow Prototype](https://docs.google.com/document/d/1_HatNe9yIf8qRVlY75JFiVGZj2jEXodZSdr3SgzkKJI/edit#)

\[2] [Corpora Metadata Requirements](https://docs.google.com/document/d/1_I2dnYuCTvO2yABR0FnLRnI8RnMhiASC5Ny4ZwDjsCI/edit#heading=h.i5w3qbht4wai)

\[3] [Personal Identification Information List](https://docs.google.com/document/d/1nlUuRiQ8Awo_QsiVFi8OWdhGONjeaXE32z5GblVCfcI/edit)

\[4] [User and System Flow Diagrams](https://drive.google.com/file/d/1pHSwhqIvhYyg1OP8z4QCeEPR1h3AQQnK/view)

## Appendix A: Change Log

### Updated 2020-07-13; Approved 2020-07-15

The Data Model has significantly changed since the original approval of this design. The following changes have been made:

- [MAJOR] The relationship between the Collection entity and the Dataset entity has been changed to 1::N (from N::M). This decision was made to make Authentication/Authorization mechanisms easier. If datasets belonged to more than one collection and the collection states/visibilities were different, then it would raise complicated questions on what the state or visibility of the dataset should be. Also, if the relationship had been N::M it would have been difficult to enforce consistency between datasets that appear in multiple collections -- we would have had to create some sort of diff tool to recognize when the dataset might have changed and kept track of that/informed the submitter that the same UUID could no longer be used, etc. tl;dr: too complicated.
- This change removes the CollectionDatasets table altogether and adds a foreign key in the Dataset table to the Collection table.
- [MINOR] Addition of `development_stage` and `development_stage_ontology` to the Dataset table to match the addition of the metadata in the Corpora Schema.
- [MINOR] Addition of `link_name` to the CollectionLinks table. This is needed in order to display on the frontend some user-friendly name that represents where the link will take you instead of displaying the raw link itself.
- [PATCH] Updating the cardinality of the relationship for the DatasetContributor table. This was simply a mistake. One dataset can appear in the table multiple times and one contributor can appear in the table multiple times. The cardinality made it seem like they could only appear once a piece.

![alt_text](imgs/updated_relational_diagram.png)

<p style="text-align: center;"><b>Figure 7</b>: Updated UML of database after applying the above mentioned changes. Live Lucidchart link is <a href=https://app.lucidchart.com/documents/edit/b88328c0-4706-4021-ac6a-74cae82704d3/0_0?shared=true>here</a>.</p>

#### 2020-07-15 Approvers

Deadline for Review: 2020-07-15

Stakeholders, please write LGTM + the date in the box next to your name after reviewing the document like this: **LGTM 2020-MM-DD.**

<table>
<tr>
 <td>Marcus Kinsella
 </td>
 <td><strong>LGTM 2020-07-15</strong>
 </td>
 <td>Trent Smith
 </td>
 <td><strong>07/15/2020</strong>
 </td>
</tr>
<tr>
 <td>Brian Raymor
 </td>
 <td><strong>LGTM 2020-07-20</strong>
 </td>
 <td>Bruce Martin
 </td>
 <td>I have concerns about the collection:dataset relationship, but those were discussed ephemerally in Slack.  The rest looks fine to me - 2020-07-15
 </td>
</tr>
</table>
