# [WIP] Authorization

**Authors:** [Brian McCandless](mailto:bmccandless@chanzuckerberg.com)

**Approvers:** [Friendly Tech Person](mailto:some.nice.person@chanzuckerberg.com), [Friendly PM Person](mailto:some.nice.person@chanzuckerberg.com), [Friendly Tech Person2](mailto:some.nice.person@chanzuckerberg.com), [Girish Patangay](mailto:girish.patangay@chanzuckerberg.com)

## tl;dr 

Authorization involves checking the visibility/owner of a collection, and handling capability URLs.

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

For this document, `obfuscated_uuid` will refer to the field in the collection's table in the
portal database.  `capability URL` will refer to any URL that embeds an `obfuscated_uuid`.  It
has the same meaning as `bearer token` in this context.

## Product Requirements

### Requirements for capability URLs.

A collection that has visibility=PRIVATE, may have an obfuscated_uuid.  If
present then the collection can be accessed using the obfuscated_uuid in place of the normal
collection uuid, which is the capability URL  APIs like `/dp/v1/collections/{collection_uuid
}`, can be accessed
 through this
capability URL: `/dp/v1/collections/{obfuscated_uuid}`, with the following differences:
 
If the capability URL is used, then no checks for visibility or user id are performed.  Anyone
with the capability URL will have access.  

The capability URL does not expire, however it can be revoked. 

An obfuscated_uuid applies to a collection, and not a dataset, and a collection can have only one.

Links to the cellxgene explorer from the portal will include the
obfuscated_uuid, so that those possessing the URL can also explore the data.
For example, instead of `/e/<dataset_name>.cxg`, 
we can use `/e/<obfuscated_uuid>/<dataset_name>.cxg`.
Or perhaps the obfuscated_uuid can be included as part of the query string.
 
## Detailed Design | Architecture | Implementation

### cellxgene Explorer

#### Handling of private datasets

This task involves checking if the visibility of the dataset is PUBLIC, or if the dataset's
 owner is the same as the current user_id.  If so, then access to the dataset can be granted.
Also, this check would need to be done on every API request so streamlining this process may be
important for performance.

The design also needs to ensure that all dataset data can be cached in CloudFront.  If the
 dataset if private, it's cached version can only be retrievable by the owner.
This will be discussed in the section on Caching.
 
The authorization determination can be done in a number of ways.  One way is to provide a backend
-only API that can answer this question.  The API can be setup so that it can only be accessed from the
private VPC, if we determine that it is important to keep a clean separation from the public
interface. 

`GET /v1/dataset/access`

**Description:** determine if the provided user_id has access to the dataset

**Parameters:**

1. user_id [REQUIRED]:  the id of the user
1. dataset_name [REQUIRED]: the name of the dataset (e.g. "pbmc3k.cxg".)

**Response:**

1. 200 OK, if the dataset is public, or the user_id is the owner.
1. 401 Unauthorized (user requested private dataset, but not logged in)
1. 403 Forbidden (user requested private dataset, but is not the owner)
1. 404 dataset not found
 
If the `/v1/dataset/access` request returns 200, then the user will be granted access.
  

#### Handling of capability URLs

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
1. dataset_name [REQUIRED]: the name of the dataset (e.g. "pbmc3k.cxg")

**Response:**

1. 200 OK, if the obfuscated_uuid is valid for the collection containing the dataset
1. 404 dataset not found or obfuscated_uuid not found (or has been revoked).

#### Caching issues

Checking the authorization on every API request is not ideal.  This increases latency on each
request, since each request would need to hit the explorer backend, and then the portal backend.  

To streamline the process, information about whether the user has access to the dataset can be
stored in a special purpose cookie. 
 
CloudFront can be configured to cache content based on cookies.  
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Cookies.html

The approach would work as follows:

1. the first time a user attempts to access a dataset (either a public dataset, a private dataset
, or a private dataset accessed through a capability URL), the authorization determination will go
through the process described in the previous sections.  If a 200 status code is returned, then a special purpose cookie is
created and returned with the results of the request.  For now, call this cookie the 
`dataset-access-cookie`.  
2. CloudFront can be configured to return separate results based on the request and the value
 of a cookie.   
3. On the next request from the client, the `dataset-access-cookie` cookie will be sent with the
 request.  If this data has previously been cached, then the cached version will be returned.
 
For this to work, the `dataset-access-cookie` needs to have different values for all datasets, 
but the same value for each user.  The value of the cookie could simply be the name of the
dataset.  However, this would be an extremely simple security system to get around, so we have
to do a bit better than that.   The value of this cookie could be the dataset name, but encrypted
with a key known only to the cellxgene backend:  `cookie_value = encrypt(dataset name, key)`

The cookie value can no longer be faked, but it could be stolen from a legitimate user and re-used.
This may not be a major concern for our use case;  the cookie value will be sent securely in the
request response, so the only major concern is stealing it from the client directly.  If needed
we could mitigate by rotating the cookie value periodically, and setting a timeout on the cache.
For example,  to rotate the cookie every day:  
`cookie_value = encrypt((dataset name, date(yyyy.mm.dd)), key)`.  
This would add some complexity, and would have the downside of invalidating the cache
every day, but solves the stolen cookie issue.   Note:  this would only needed for private
collections.  Another less severe option is to rotate the
key when we do our weekly upgrades, since we invalidate the cache at that time anyways.

If there is a CloudFront cache miss, then the cookie is sent in the request to the cellxgene
explorer backend.  The `dataset-access-cookie` can inform the backend that the server has access
to a particular dataset.  If that dataset is the one being requested, then access is granted.
The response will the be cached for the next matching request.

The scheme described in this section only gives a user access to one dataset at a time.  If the
user is working on dataset A, then switches to dataset B, then CloudFront will have a cache miss
, and the authorization process will be done to make sure the user has access to dataset B.
In general this is not a problem.  The first API requested is the /schema and /config endpoint
, which are very small amount of data.  Later requests like the layout and annotations will have an
opportunity to hit the cache.

Other issues:

If the owner of a dataset revokes the capability URL, then the CloudFront cache should be
invalidated for paths that include the obfuscated_uuid, otherwise someone who previously
accessed the data could get a cache hit from CloudFront if they accessed it after it had been
revoked.   If invalidating the cache added to much complexity, then the issue eventually fixes
itself when the cache timeout.  We could also set an expiration time on the `dataset-access
-cookie`, which doesn't solve the issue, but adds one more opportunity to prevent accidental access.


Alternatives:

A solution to authorization and CloudFront could be implemented in Lambda@Edge.  I briefly looked into
this, and thought it would be more complicated than the above proposal.  Also, it would still have to find
a way to access to backend of the cellxgene portal to determine if the user has access to the
 dataset.
  
  
#### Community releases vs hosted cellxgene.


There should be a configuration option to set the authorization type, such as how we do with
authentication.  The authorization type we use for CZI hosted cellxgene possibly may be better
implemented as a plugin, using our plugin architecture, so as not to make public the
details of the implementation.

In the scenario described in this document, the authorization component will need to answer the
question: does the current user have access to the dataset at this path?  It will also need to
return the dataset name from the path, since in the case of the obfuscated_uuid feature, the name 
is embedded in the path, and needs to be extracted.  
 

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
Or maybe they are the owner, but not logged into right now, so it is reasonable they have the URL;
Or maybe they have a capability URL that has been revoked.
Should we inform the user that they do not have access, or return a 404 error?
