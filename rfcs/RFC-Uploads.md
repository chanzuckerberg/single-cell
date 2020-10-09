# User Uploads

## For the cellxgene Data Portal

**Authors:** [Trent Smith](mailto:trent.smith@chanzuckerberg.com)

**Approvers:** [Arathi Mani](mailto:arathi.mani@chanzuckerberg.com), [Timmy Huang](mailto:thuang@chanzuckerberg.com),
[Marcus Kinsella](mailto:mkinsella@chanzuckerberg.com), [Eduardo Lopez](mailto:elopez@chanzuckerberg.com), [Brian Raymore](mailto:braymor@chanzuckerberg.com)
[Sara Gerber](mailto:sara.gerber@chanzuckerberg.com)

## tl;dr

A user can upload files of any size to Data Portal using either a shared link from their cloud provider or directly from their browser.

## Glossary

- CSP(Cloud Storage Provider) - Cloud storage services such as Dropbox or Google Drive.
- Shared Link - A link from a CSP that gives the bearer of that link access to private data.
- DP(Data Portal) - The cellxgene Data Portal.
- Small File - a file < 5 GB in size.
- Large File - a file > 5 GB in size.

## Problem Statement | Background

One of the primary goals of the DP is to allow users to upload their own files. These files consist of data sets as well
as attestation files. These data sets can vary in size from a few MB to several GB. Users may want to upload from their local machine or their CSP. For users that want to upload from their local machine, they must be able to resume an
upload in the event their connection is interrupted. For users that want to upload files from their CSP, they can provide a shared link from their CSP that is used to upload directly to S3.

This specification does not cover the validation of the data or the process for a user to complete a submission.

## Product Requirements

1. A user can upload a file to DP using a shared link from their Dropbox.
1. A user can upload a file to DP using a shared link from their Google Drive.
1. A user can upload a file from their local machine using the DP browser.
1. A user can resume an upload without completely restarting.
1. The user must be authenticated before uploading.
1. A user who has not yet agreed to the Portal policies must not be able to upload their file.

### Nonfunctional requirements

1. Cloud to Cloud uploads will retry if they fail. Retry at least 5 times with an exponential back off.
1. A user can upload a single file up to 30GB in size.
1. Shared links referring to more than file will be rejected

## Detailed Design | Architecture | Implementation

![Upload Design](https://app.lucidchart.com/publicSegments/view/a74f747f-d071-41ea-949f-52f38a98747a/image.png)
Figure 1: The Upload Design

### Components

This section describes the purpose of components that make up the upload design.

#### Data Portal Browser App

The DP Browser App is the frontend of the system. It is the primary method users have to interact with the Data Portal.
It will be used to authenticate the user and interact with the DP Backend to upload files. It will also use AWS JS SDK to
upload files from local storage to AWS. These features will extend the existing DP Frontend.

#### Data Portal Backend

The DP Backend is responsible for handling API requests. It will generate pre-signed URLs and access tokens
for the DP Browser App to facilitate the uploading of files. It will also enqueue shared links for downloading.
This additional functionality can be built into the existing DP Backend Lambda. 

The backend will also be responsible for performing validation of the file being uploaded. The validation is
performed during the initial request to upload. 

#### Download Queue

The Download Queue is an [AWS SQS](https://aws.amazon.com/sqs/) that will track pending and in-progress download jobs.
The download jobs are used by the EC2 instances to know what files to download before uploading them to S3.

##### Download Job Entry

These are the fields that will be in the download job placed in the queue:

- Link - a to the link to download. Use to retrieve the file from CSP.
- submission_id - identifies the submission. Used to determine the storage location.
- dataset_id - identifies the dataset. Used to determine the storage location
- file_name - the name of the file being downloaded

#### AWS STS

The AWS STS service will be used to generate temporary credentials used to upload a file to S3. The backend
server will use the STS [assume role](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html)
functionality to generate a token with temporary access to upload to S3. This token is referred to as an access token.

The access token generated and provided grants direct access to the S3 bucket for uploading. This will allow
the browser APP to use the AWS JS SDK to perform the multipart uploads to S3 for large files. A backend server will be used
to generate the token and additional sessions policies will be applied to the token to further restrict access

According to AWS:

> The resulting session's permissions are the intersection of the role's identity-based policies and the session policies.

The IAM policies bellow will be combined to limit the access of the token generated to only allow the bearer to upload the
file in the session policy resource. The session policy will be generated at the time of the request and will use the
asset_uuid generated by the server and the data set_uuid provided in the request.

##### Assumed Role IAM Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListMultipartUploadParts",
        "s3:PutObject",
        "s3:AbortMultipartUpload"
      ],
      "Resource": ["arn:aws:s3:::upload-bucket/"]
    }
  ]
}
```

##### IAM Session Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListMultipartUploadParts",
        "s3:PutObject",
        "s3:AbortMultipartUpload"
      ],
      "Resource": [
        "arn:aws:s3:::upload-bucket/{submission_id}/{dataset_id}/{file_name}"
      ]
    }
  ]
}
```

#### EC2

The EC2 instance will be used to download files from a CSP and upload it to our Upload bucket.

#### Upload Bucket

The upload bucket is a separate bucket or separate bucket location within our main bucket. It is used to stage files
before moving to the main bucket.

S3 event can be used to trigger additional processing such as validation, once the upload has completed. The additional
processing is outside the scope of this RFC.

### Data Portal APIs

The DP Browser App will use these endpoints to interact with the Backend using these APIs. This section describes the new
endpoints and how they are used. These are the new API endpoints needed to complete an upload.

These APIs replace the [PUT /v1/submission/{project_uuid}/file](https://docs.google.com/document/d/1d8tv2Ub5b3E7Il85adOAUcG8P05N6UBZJ3XbhJSRrFs/edit#heading=h.3hln6w2kzoyt)
section.

#### PUT submission/{submission_id}/upload/link

An authenticated user can upload a file from a shared link to a data set in their submission.

**Request:**

| Parameter     | Description                                               |
| ------------- | --------------------------------------------------------- |
| Link          | a shared link to the file                                 |
| submission_id | identifies the submission                                 |
| attestation   | A boolean value indicating if the file is for attestation |
| _Optional_ dataset_id | identifies the dataset being uploaded. Used to check status of upload|

**Response:**

| Key        | Description                                                                |
| ---------- | -------------------------------------------------------------------------- |
| dataset_id | identifies the data set the file will be associated with after validation. |
| status     | Provides the current status of the upload. The status can be 

**Error Responses:**

| Code | Description                                                                                                             |
| ---- | ----------------------------------------------------------------------------------------------------------------------- |
| 401  | if dataset_id or submission_id does not exist, or if the user does not own the submission or upload in-progress upload. |
| 400  | if the file type is invalid                                                                                            |

#### POST submission/{submission_id}/upload/file

An authenticated user can upload a file directly from their local machine to a data set in their submission.

Asset_uuid can be included to retrieve a new access token if you were unable to complete the upload before the token
expired. An error will be returned if the upload has already been completed. The user must own the submission and be
the one who initiated the upload to retrieve an additional access token.

If the size of the file being upload is <5GB, then a pre-signed URL will be returned.
If the size of the file is >5GB, then an access token will be returned.

**Request:**

| Parameter             | Description                                                                |
| --------------------- | -------------------------------------------------------------------------- |
| File size             | the size of the file to be uploaded                                        |
| File Name             | the name of the file being uploaded                                        |
| submission_id         | identifies the submission                                                  |
| attestation           | A boolean value indicating if the file is for attestation                  |
| _Optional_ dataset_id | identifies the dataset being uploaded. Used if resuming an existing upload |

**Response:**

| Key           | Description                                                   |
| ------------- | ------------------------------------------------------------- |
| Access Token  | allows the user to upload their file to S3 in multiple parts. |
| Presigned URL | allows the user to upload their file to S3 in a single part.  |
| dataset_id    | identifies the name of the new file created.                  |

**Error Responses:**

| Code | Description                                                                                                             |
| ---- | ----------------------------------------------------------------------------------------------------------------------- |
| 401  | if dataset_id or submission_id does not exist, or if the user does not own the submission or upload in-progress upload. |
| 400  | if the file type is invalid                                                                                            |

### Upload Flows

There are two methods for uploading data to the DP. They are Cloud to Cloud and Local to Cloud. The follow
sections details the user interactions required, and how the Data Portal backend will facilitate the upload.

#### Cloud to Cloud Upload Flow

Cloud to cloud upload requires the user to provide a shareable link for a file on their CSP from which DP can upload
the file to the S3 bucket. Now to walk through the cloud to cloud flow, see figure 1.

![Cloud to Cloud](https://app.lucidchart.com/publicSegments/view/e1e72143-c108-4c7f-bf6c-053d98256c1f/image.png)
**Figure 2:** Architecture of a user uploading from their cloud to our cloud.

##### 1. Get Share Link

The user must generate a sharable link from their CSP. Both Google Drive and Dropbox support the generation of
sharable links. Only sharable links should only point to a single file, not a folder or submission of files.

##### 2. Share Link

Using the browser App, make a \_POST {submission_id}/upload/link_request to upload the file
by providing the sharable link in the request body. The response will contain the dataset_id, that can be used to
check the status of the upload

##### 3. Sanitize

To prevent [server side request forgery](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery) the
incoming shared link will be sanitized before queuing for download. The links generated by CSP are often not direct
links to the files and often need to be parsed into the correct shape for downloading. The correct URL will be generated
as part of the sanitization process. The backend that runs this should also be given minimal privileges to further protect from
server-side request forgery.

###### How to download from a shared link

- Dropbox <https://help.dropbox.com/files-folders/share/force-download>
- Google Drive <https://www.labnol.org/internet/direct-links-for-google-drive/28356/>

##### 4. _Optional_ Queue

This part is optional and depends on the load expected and server type selected. The queue stores the links that need to
be processed. It buffers the downstream servers from download requests. This allows us to spin up servers as needed.
It also gives us the ability to retry a download if one of the servers fails to download or upload the file.

###### Why might the download fail

- The downloading server ran out of storage.
- The downloading server crashed.
- The CSP experience and outage
- The user has reached their daily bandwidth allowance from their CSP. [more info](https://help.dropbox.com/files-folders/share/banned-links)

##### 5. Start EC2 Instance to Download File

Spin up EC2 instances with enough storage to download the whole file. TODO: See if spot instances could be used.

##### 6. Stream File

The shared link is used to stream the file from one cloud to another. The most time will be spent streaming the file 
from the CSP to S3. Most CSPs offer a way to validate the integrity of a download
by providing a hash of the data, for example [Dropbox Content Hash](https://www.dropbox.com/developers/reference/content-hash).
As the data is streaming from the CSP, the hash of the file will be calculated as it arrives, and again as it's 
uploaded to S3. The [Content-MD5 header](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObject.html#API_PutObject_RequestSyntax)
will be used when uploading to S3 to verify the integrity of each chunk. The AWS SDK will be useful in simplifying the
upload to S3.

Once the upload is complete, the download job will be removed from the queue. If the final hash does not match, then 
the upload process will be restarted. The retry logic will be an exponential back off, with a max retry of 5 times.

> QUESTION? Do we need to tell the user or the DB when the file is done do uploading? This makes me think we need an endpoint
> to finalize the upload. The endpoint will look for the expected file, and if it's found, update the database with the file
> info.

#### Local to Cloud Flow

Local to cloud upload will allow users to upload files from their local machine to the cloud. Uploads happen though the
DP browser App directly to S3. For small files <5GB, uploads can be accomplished using pre-signed URLs. For large files >5GB
a multipart upload must be performed. Now to walk through the local to cloud flow, see figure 2.

![Local to Cloud](https://app.lucidchart.com/publicSegments/view/80361b71-5317-478b-921e-0ead5152c865/image.png)
**Figure 3:** Architecture for uploading from a local machine to our cloud.

##### 1. Upload File

Using the browser App, a file is selected for upload and a request is sent to the DP backend upload the file.

##### 2. Request Token or Presigned URL

Depending on the size of the file a different response will be returned. If an access token is returned, a
multipart upload is started using the AWS SDK. If a pre-signed URL is returned an upload is started using the URL.

##### 3. Upload Directly S3

The implementation uses an access token and the AWS JS SDK to upload the file directly to S3. The hash of each chunk must be  
calculated locally and supplied in the [Content-MD5 header](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObject.html#API_PutObject_RequestSyntax)
as part of Put Object request. This will verify the integrity of each chunk during the upload. If the hash does not match
the chunk will be rejected.

##### User Upload Limitation

S3 is the CZI CSP where uploaded datasets are stored. Since S3 allows small files to be uploaded without using multipart
upload, our implementation can generate a single pre-signed URL when securely uploading small data sets.

For large files, AWS S3 requires that the files be uploaded using their multipart upload process. This requires multiple
requests to S3, and cannot be completed with a single pre-signed URL. In this case, a temporary security token is generated.
The token grants the bearer directs access to the S3 bucket. This allows the browser application to use
the AWS JS SDK to upload a large file in multiple parts with ease. It is important to limit the permissions of the token
to only allow it to upload the expected file. There is also a maximum time to live of one hour for the token, therefore if
the upload is not complete before then, a new token must be issued.

##### Cleanup Multipart Uploads

In some cases, a multipart upload may fail to complete and be left partially uploaded to s3. AWS will retain these partial
uploads and charge for the storage. These charges can be mitigated by applying an S3 bucket life cycle rule to clean up
incomplete multipart uploads, see [link](https://aws.amazon.com/blogs/aws/s3-lifecycle-management-update-support-for-multipart-uploads-and-delete-markers/).

### Test plan

1. Verify that a large file can be uploaded using a shared link from Dropbox or Google Drive.
1. Verify that a large file can be upload from a users local machine
1. Verify that a small file can be uploaded using a shared link from Dropbox or Google Drive.
1. Verify that a small file can be upload from a users local machine
1. Verify the access token provided to the user can only be used to upload that single file.
1. Verify a local download can be resumed.
1. Verify the download queue is not cleared until the file is store in S3.

### Monitoring and error reporting

- Monitor the length of the Download queue. If it gets too long, then increase the number of EC2 instances used.
- Log downloading failure on the EC2 instances.

## [Optional] Alternatives

### Download using SMB

Dropbox and Google drive support syncing files to a server using their desktop application. This used SMB protocol
to sync your local storage with what is in the cloud. This will allow a user to share a file with us by adding us
to their shared folder. This would cause the file to sync to our system. This approach would solve the download problem,
and at the same time introduce more uploading and cleanup problems. The download to our server would be asynchronous,
which would mean the server needs to be up 100% of the time. Instruction must be provided to users on how to share
their files with our account. The file names would be controlled by the user until they are synced into our server.
From that point, a mapping of files to their file name uploaded to S3 would be needed. Given a cluster of servers, I'm not
certain if the SMB protocol would duplicate the files across all of the servers or if the servers can be instructed which
files to clone.

## References

Any relevant documents or prior art should be clearly listed here. Any citation of figures or papers should also be clearly listed here. Of course, they can also be linked throughout the document when directly referenced.

- [S3 IAM Permissions](https://docs.aws.amazon.com/AmazonS3/latest/dev/list_amazons3.html)
- [AWS S3 Multipart Upload Permissions](https://docs.aws.amazon.com/AmazonS3/latest/dev/mpuAndPermissions.html)
- [Original Ticket](https://app.zenhub.com/workspaces/single-cell-5e2a191dad828d52cc78b028/issues/chanzuckerberg/single-cell/25)
- [HCA Upload Service](https://docs.google.com/document/d/1PiO-0ThE7GxQpw7XqkK8P9d8E8xP8K9ssMCR0w4WTd4/edit) - lot of good ideas and things to consider.
- [Corpora High Level Architecture](https://docs.google.com/document/d/1d8tv2Ub5b3E7Il85adOAUcG8P05N6UBZJ3XbhJSRrFs/edit#heading=h.3hln6w2kzoyt)
- [AWS-S3-upload-integrity](https://stackoverflow.com/questions/42208998/aws-s3-upload-integrity)
