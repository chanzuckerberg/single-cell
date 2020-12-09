# [WIP] Authorization

**Authors:** [Brian McCandless](mailto:bmccandless@chanzuckerberg.com)

**Approvers:** [Friendly Tech Person](mailto:some.nice.person@chanzuckerberg.com), [Friendly PM Person](mailto:some.nice.person@chanzuckerberg.com), [Friendly Tech Person2](mailto:some.nice.person@chanzuckerberg.com), [Girish Patangay](mailto:girish.patangay@chanzuckerberg.com)

## tl;dr 

Authorization involves checking the visibility/owner of a collection, and handling obfuscated URLs.

## Problem Statement | Background

From the ticket: https://github.com/chanzuckerberg/corpora-data-portal/issues/750  

The Portal and Explorer now both implement authentication using Auth0. 
And, when a collection is private, the Portal verifies that the requester is authenticated and is the owner of the collection.
We have a couple more authorization requirements we need to design:

1. The Explorer needs to know about authz. Currently, it checks the user's identity so it can
 associate user annotations with the correct user. 
 But it also needs to apply the authorization
  logic that's currently in the Portal: only let the owner see private data.
2. We have a long-lived bearer token (aka obfuscated uuid) authz use case that needs to work in
 Portal and Explorer.

Right now all the authz information lives in the Portal, so the Explorer has no idea what an "owner" is. 
Also, the design needs to not harm performance in the explorer: things like caching in the
instances and the CDN should still work correctly.


## [Optional] Product Requirements

### Requirements for obfuscated uuid

From the 0001-data-portal-architecture, under Publishing a Collection Privately:

    Publishing a collection privately runs the full data processing pipeline on the given collection
    details and datasets and delivers a URL where the submitted may view their generated collection
    page as well as their dataset(s) in cellxgene. This URL, under the hood, will point to a
    permanent URL but will be unindexed and obfuscated (though sharable)....

A collection that has visibility=PRIVATE, may have an obfuscated_uuid.  If
present then the collection can be accessed using the obfuscated_uuid in place of the normal
collection uuid.  API's like `/dp/v1/collections/{collection_uuid}`, can be accessed through this
URI: `/dp/v1/collections/{obfuscated_uuid}`, with the following differences:
 
If the obfuscated_uuid is used, then no checks for visibility or user id are performed.  Anyone
with the obfuscated_uuid will have access.  

The obfuscated_uuid does not expire, however it can be revoked.

An obfuscated_uuid applies to a collection, and not a dataset, and a collection can have only one.

Links to the cellxgene explorer from the portal will need to include some information about the
obfuscated uuid, so that those possessing the URL can also explore the data.
For example, instead of `/e/<dataset_name>.cxg`, we use `/e/<obfuscated_uuid>/<dataset_name>.cxg`.
Or perhaps the obfuscate_uuid can be included as part of the query string.
 
## Detailed Design | Architecture | Implementation

### cellxgene Explorer

#### Handling of private datasets

This task involves checking if the visibility of the dataset is PUBLIC, or if the dataset's
 owner is the same as the current user_id.  If so, then access to the dataset can be granted.
Also, this check would need to be done on every API request so streamlining this process may be
important for performance.

To streamline the process, information about whether the user has access to the dataset can be
stored in the session cookie.  The session cookie stores information as dictionary.  I propose a
key named "dataset_access_grant", and a value that is a dictionary with the keys "dataset" and
"expiration".  The check for the session cookie might look like this.

```python
def has_dataset_access_grant(dataset_location):
    value = session.get(“dataset_access_grant”)
    if value is None:
        return False
    if value.get(“dataset”) == dataset_location:
       expiration = value.get(“expiration”)
       if expiration is not None and expiration < time.time():
           # access has been granted
          return True
    else:
       return False
```
 
If there is not a current access grant, then we must determine if this user has access to the
dataset. Note, cellxgene explorer knows only the dataset name, not the dataset uuid.

This determination can be done in a number of ways.  One way is to provide a backend-only API
that can answer this question.  The API can be setup so that it can only be accessed from the
private VPC, if we determine that it is important to keep a clean separation from the public
 interface. 

`GET /v1/dataset/access`

**Description:** determine if the provided user_id has access to the dataset

**Parameters:**

1. user_id [REQUIRED]:  the id of the user
2. dataset_name [REQUIRED]: the name of the dataset (e.g. "pbmc3k.cxg".)

**Response:**

1. 200 OK, if the dataset is public, or the user_id is the owner.
2. 401 Unauthorized
3. 404 dataset not found
 
If the `/v1/dataset/access` request returns 200, then the user will be granted access.  The
session cookie key `dataset_access_grant` will be set with a new dataset name and expiration time.  
The time should be chosen so that the check only needs to be done occasionally.  
Probably anywhere from 30 seconds to 30 minutes is fine. 


#### Handling of obfuscated datasets

This discussion assumes that the obfuscated path has this form:  `/e/<obfuscated_uuid>/<datset_name>
.cxg`, although other options are possible.

In this way, it is easy to separate out the two components from the path.

For obfuscated datasets, the authorization portion will go through an API to check the
obfuscated_uuid and dataset.  It will ensure that the obfuscated_uuid belongs to a collection
that contains the dataset, and has not been revoked.
  
`GET /v1/dataset/access_obfuscated`

**Description:** determine if the obfuscated_uuid provides access to the dataset

**Parameters:**

1. obfuscated_uuid [REQUIRED]:  the collection's obfuscated_uuid
2. dataset_name [REQUIRED]: the name of the dataset (e.g. "pbmc3k.cxg")

**Response:**

1. 200 OK, if the obfuscated_uuid is valid for the collection containing the dataset
2. 401 Unauthorized
3. 404 dataset not found or obfuscated_uuid not found.

#### Community releaseas vs hosted cellxgene.

There should be a configuration option to set the authorization type, such as how we do with
authentication.  The authorization type we use for CZI hosted cellxgene possibly may be better
implemented as a plugin, using our plugin architecture, so as not to make public the
details of the implementation.

In the scenario described in this document, the authorization component will need to answer the
question: does the current user have access to the dataset at this path?  It will also need to
return the dataset name from the path, since in the case of the obfuscated_uuid feature, the name 
is embedded in the path, and needs to be extracted.  
 
#### Caching issues

Checking authorization requires access to the cellxgene explorer backend, which accesses the
cellxgene portal backend.
 
I need to do some research to determine how to force the authorization check on each API request
to the explorer, but also caches copies of the data on cloudfront.

### cellxgene Portal

The `/v1/dataset/access/` and `v1/dataset/access_obfuscated` APIs will be implemented in the portal
 code base, since they will need access to the portal database.

To implement this efficiently, it may be useful to create two database views: One that maps
`dataset.name` to `collection.owner` and `collection.visibility`.  And one that can efficiently
determine if a given `collection.obfuscated_uuid` is associated with a collection that contains a
 given `dataset.name`.
 
### UX issues

#### explorer

1. What should be displayed if the user is not authorized to see a dataset?  
Say a user goes to ".../e/dataset.cxg", but that dataset is private and they are not the owner; 
Maybe they are the owner, but not logged into right now, so it is reasonable they have the URL. 
All the explorer backend APIs would return 401 errors in this case. 
The explorer frontend, by default, would just show "Error" in red. 
But we may want to say something nicer like "Sorry, you don't have access to this dataset". 
A similar situation could arise for revoked capability urls.
Another possibility is to return a "404" error if the user does not have access to a dataset.
 
