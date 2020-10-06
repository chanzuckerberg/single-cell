# User Uploads


## For the cellxgene Data Portal

**Authors:** [Trent Smith](mailto:trent.smith@chanzuckerberg.com)

**Approvers:** [Friendly Tech Person](mailto:some.nice.person@chanzuckerberg.com), [Friendly PM Person](mailto:some.nice.person@chanzuckerberg.com), [Friendly Tech Person2](mailto:some.nice.person@chanzuckerberg.com), [Girish Patangay](mailto:girish.patangay@chanzuckerberg.com), [Sara Gerber](mailto:sara.gerber@chanzuckerberg.com)

## tl;dr

A user can upload files of any size to Data Portal using either a shared link from their cloud provider, or directly from their browser.

## Glossary

- CSP(Cloud Storage Provider) - Cloud storage services such as Dropbox or Google Drive.
- Shared Link - A link from a CSP that give the bearer of that link access to private data.
- DP(Data Portal) - The cellxgene Data Portal.
- Small File - a file < 5 GB in size.
- Large File - a file > 5 GB in size.

## Problem Statement | Background

One of the primary goals of the DP is to allow users to upload their own data sets. These data sets can vary in
size from a few MB to several GB. User may want to upload from their local machine or from their CSP. For
user that want to upload from their local machine, they must be able to resume an upload in the event their connection
is interrupted. For users that want to upload data sets from their CSP, they are able to provide a shared link from
their CSP that we can used to upload directly to our storage.

## Product Requirements

1. A user can upload a file to DP using a shared link from their CSP.
1. A user can upload a file from their local machine using the DP browser.
1. A user can resume an upload with out completely restarting.
1. The user must be authenticated before uploading.


### Nonfunctional requirements
Nonfunctional requirements are not directly related to the user story but may instead reflect some technical aspect of the implementation, such as latency. [Here's](https://en.wikipedia.org/wiki/Non-functional_requirement) a more comprehensive list of nonfunctional requirements.


1. Cloud to Cloud uploads will retry if they fail.
1. A user can upload a single file up to 30GB in size.

## Detailed Design | Architecture | Implementation
![Upload Design](https://app.lucidchart.com/publicSegments/view/a74f747f-d071-41ea-949f-52f38a98747a/image.png)
Figure 1: The Upload Design

### Data Portal Browser App
The DP Browser App will be used authenticate the user, and interact with the DP Backend to upload files.
It will also use AWS JS SDK to upload files from local storage to AWS. These feature will extend the existing DP
Frontend.

### Data Portal Backend
The DP Backend is responsible for handling users requests. It will generated presigned URL and access tokens 
for the DP Browser App to facilitate the uploading of files. It will also enqueue shared links for downloading.
This additional functionality can be built into the existing DP Backend Lambda.

### Download Queue
The Download Queue is an [AWS SQS](https://aws.amazon.com/sqs/) that will track pending and in progress download jobs.

#### Download Job Entry
These are the fields that will be in the download job placed in the queue:
- Link - a to the link to download. Use to retrieve the file from CSP.
- collect_id - identifies the collection. Used to determine the storage location.
- dataset_id - identifies the dataset. Used to determine the storage location
- asset_id - identifies the asset. Used to determine the storage location, and file name.
- file size - the size of the asset to download. Used to determine the download method, and EC2 instance size.

### Data Portal APIs
The DP Browser App will use these endpoint to interact with the Backend using these APIs. These are the new
API needs to complete an upload.

#### POST {collection_id}/{data set_id}/upload/link
An authenticated user can send a request to upload a file from a shared link to their data set in their collection.

##### Request
- Link - a shared link to the file
- collection_id - identifies the collection
- data set_id - identifies the data set

##### Response
- Asset_uuid - identifies the name of the new file created.

#### POST {collection_id}/{dataset_id}/upload/file
An authenticated user can send a request to upload a file directly from their local machine to their data set in their 
collection.

##### Request: POST {collection_id}/{dataset_id}/upload/file
- File size - the size of the file to be uploaded
- collection_id - identifies the collection
- data set_id - identifies the data set

##### Response: POST {collection_id}/{dataset_id}/upload/file
- Access Token - allows the user to  upload their file to S3 in multiple parts. 
- Presigned URL - allows the user to upload their file to S3 in a single part.
- asset_uuid - identifies the name of the new file created.

#### POST /dp/v1/dataset/{dataset_uuid}/asset/{asset_uuid}/upload
Used to request an access token to continue uploading a file to S3 by an authenticated user who owns the data set. An 
error is returned if the file upload has been completed already.

##### Request: POST /dp/v1/dataset/{dataset_uuid}/asset/{asset_uuid}/upload
- data set_id - identifies the data set.
- asset_id - identifies the asset.

##### Response: POST /dp/v1/dataset/{dataset_uuid}/asset/{asset_uuid}/upload
- Access Token - allows the user to  upload their file to S3 in multiple parts. 

### AWS STS
The AWS STS service will be used to generate temporary credential the user can use to upload a file to S3. The backend
server will use the STS [assume role](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) 
functionality to generate a token with temporary access to upload to S3. According to AWS:

> The resulting session's permissions are the intersection of the role's identity-based policies and the session policies.

The IAM policies bellow will be combined to limit the access of the token generate to only allow the bearer to upload the 
file in the session policy resource. The session policy will be generated at the time of the request, and will use the
asset_uuid generated by the server and the data set_uuid provided in the request.

#### Assumed Role IAM Policy
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
        "arn:aws:s3:::upload-bucket/"
      ]
    }
}
```

#### IAM Session Policy
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
        "arn:aws:s3:::upload-bucket/{data set_uuid}/{asset_uuid}"
      ]
    }
}
```


### EC2
The EC2 instance will be use to download files from a CSP and upload to our Upload bucket. 

### Upload Bucket
The upload bucket is a separate bucket or separate bucket location within our main bucket. It is used to stage files 
before moving to the main bucket.

S3 event can be use to trigger additional processing such as validation, once an upload has completed.

### Cloud to Cloud Upload Flow
Cloud to cloud uploads will allow a user to provide a sharable link from their CSP to the file they wish to upload,
which DP will used to upload the file to our S3 bucket. Now to walk though the cloud to cloud flow, see figure 1.

![Cloud to Cloud](https://app.lucidchart.com/publicSegments/view/e1e72143-c108-4c7f-bf6c-053d98256c1f/image.png)
**Figure 2:** Architecture of a user uploading from their cloud to our cloud.

#### 1. Get Share Link
The user will generate a sharable link from their CSP. Both Google Drive and Dropbox support the generation of
sharable links. Only sharable links should only point to a single file, not a folder or collection of files.

#### 2. Share Link
Using the browser App, the user will make a *POST {collection_id}/{dataset_id}/upload/link* request to upload the file 
by providing the sharable link in the request body. The response will contain the asset uuid, that the user can use to 
check the status of the upload

#### 3. Sanitize
To prevent [server side request forgery](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery) the
incoming shared link will be sanitized before queuing for download. The links generated by CSP are often not direct
links to the files and often need to be parsed into the correct shape for downloading. The correct URL will be generated
as part of sanitization. The backend that runs this should also be given minimal privileges to further protect from 
server side request forgery.

##### How to download from a shared link
- Dropbox https://help.dropbox.com/files-folders/share/force-download
- Google drive https://www.labnol.org/internet/direct-links-for-google-drive/28356/

#### 4. _Optional_ Queue
This part is optional and depends on load we expect and server type we select. The queue stores the links that need to
be processed. It buffers the down stream servers from download requests. This allows us to spin up servers as needed.
It also gives us the ability to retry a download if one of servers fails to download or upload the file.

##### Why might the download fail?
- The downloading server ran out of storage.
- The downloading server crashed.
- The CSP experience and outage
- The user has reached their daily bandwidth allowance from their CSP. [more info](https://help.dropbox.com/files-folders/share/banned-links)

#### 5. Start EC2 Instance to Download File
Spin up EC2 instances with enough storage to download the whole file. TODO: See if spot instances could be used.

#### 6. Download the File
The most time will be spent here, downloading the file.

#### 7. Upload to S3
If the file is >5GB in size we can upload the file directly to s3 without writing to disk. For files <5GB, the file must
 be completely downloaded before it can be uploaded to s3. The AWS SDK will be used to upload large files in parts.
 Once upload is complete, and the download job will be removed from the queue.

> ??? Do we need to tell the the user or the DB when the file is done do uploading. This makes me think we need an endpoint
to finalize the upload. The endpoint will look for the expected file, and if it's found, update the database with the file
info.

### Local to Cloud Flow
Local to cloud upload will allow users to upload files from their local machine to the cloud. Uploads happen though the 
DP browser App directly to S3. For small files <5GB, uploads can be accomplished using presigned URLS. For large files >5GB 
the user must perform a multipart upload. Now to walk though the local to cloud flow, see figure 2.

![Local to Cloud](https://app.lucidchart.com/publicSegments/view/80361b71-5317-478b-921e-0ead5152c865/image.png)
**Figure 3:** Architecture of a user uploading from their local machine to our cloud.

#### 1. Upload File
Using the browser App, the user select a file to upload and send a request to upload the file.

#### 2. Request Token or Presigned URL
Depending on the size of the file a different response will be returned. If the file is >5GB

##### User Upload Limitation 
S3 is our CSP, and is where a user's uploaded files will eventually end up. S3 will allow small files to be upload without
using multipart upload. This allows us to generate a single presigned URL that the user can use to securely upload their
files.

For large files, AWS S3 requires that the files be uploaded using their multipart upload process. This requires multiple
requests to S3, and cannot be completed with a single presigned URL. In this case, a temporary security token is generated
for the user. Token will grants the bearer directs access to the S3 bucket. This allows the browser application to use
the AWS JS SDK to upload a large file in multiple parts with ease. It important to limit the permissions of the token
to only allow it upload the expected file. There is also a maximum time to live of one hours for the token, therefore if
the upload is not complete before then, a new token must be issued.

### Access Token
The access token generated and provided to the user grants direct access to the S3 bucket for uploading. This will allow
the browser APP to use the AWS JS SDK to perform the multipart uploads to S3 for large files. A backend sever will be used
to generate the token and additional sessions policies will be applied to the token to further restricts accoess

### Cleanup Multipart Uploads
In some cases a multipart upload may fail to complete and be left partial uploaded to s3. AWS will retain these partial
uploads and charge for the storage. These charges can be mitigated by applying an S3 bucket life cycle rule to clean up
incomplete multipart uploads, see [link](https://aws.amazon.com/blogs/aws/s3-lifecycle-management-update-support-for-multipart-uploads-and-delete-markers/).

### Completing an Upload
Once the file has been uploaded to our bucket additional process may take place. The bucket can be configure to emit events
when a file has been uploaded that can trigger additional processing. The additional processing is outside the scope of this 
RFC.

### Test plan
1. Verify that a large file can be uploaded using a shared link from Dropbox or Google Drive.
1. Verify that a large file can be upload from a users local machine
1. Verify that a small file can be uploaded using a shared link from Dropbox or Google Drive.
1. Verify that a small file can be upload from a users local machine
1. Verify the access token provided to the user can only be used to upload that single file.
1. Verify a local download can be resumed.
1. Verify the download queue is not cleared until the file is store in S3.

### Monitoring and error reporting

- Monitor the length of the Download queue. If it gets too long we may need to increase the number of EC2 instance
- Log downloading failure on the EC2 instances.

## [Optional] Alternatives

### Download using SMB
Dropbox and Google drive support syncing files to a server using using their desktop application. This used SMB protocol
to sync your local storage with what is in the cloud. This will allow a user to share a file with us using by adding us
to their shared folder. This would cause the file to sync to our system. This approach would solve the download problem,
and at the same time introduce more uploading and cleanup problems. The download to our server would be asynchronous, 
which would mean the server needs to be up 100% of the time. We would need to provide instruction to users on how to share
their files with our account. The file names would be controlled by the user until they are synced into our server. 
From that point we would need to map the files to their file name uploaded to S3. If we had a cluster of servers I'm not
sure if the SMB protocol would duplicate the files across all of the server or if we can tell the servers which files to
clone.


## References

Any relevant documents or prior art should be clearly listed here. Any citation of figures or papers should also be clearly listed here. Of course, they can also be linked throughout the document when directly referenced.
- [S3 IAM Permissions](https://docs.aws.amazon.com/AmazonS3/latest/dev/list_amazons3.html)
- [AWS S3 Multipart Upload Permissions](https://docs.aws.amazon.com/AmazonS3/latest/dev/mpuAndPermissions.html)
- [Original Ticket](https://app.zenhub.com/workspaces/single-cell-5e2a191dad828d52cc78b028/issues/chanzuckerberg/single-cell/25)
