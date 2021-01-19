# Authentication in Data Portal and Cellxgene <img style="float: right;" src="./imgs/cat.png" width="200">

**Authors:** [Brian McCandless](mailto:bmccandless@chanzuckerberg.com)

**Approvers:**
[Timmy Huang](mailto:thuang@chanzuckerberg.com),
[Trent Smith](mailto:trent.smith@chanzuckerberg.com),
[Arathi Mani](mailto:arathi.mani@chanzuckerberg.com),
[Colin Megil](mailto:colin.megil@chanzuckerberg.com),
[Eduardo Lopez](mailto:elopez@chanzuckerberg.com),
[Ambrose Carr](mailto:acarr@chanzuckerberg.com)

## tl;dr

The initial version of this document described the authentication process
for Data Portal and Cellxgene Explorer to address the immediate needs of M1, which is to support user
annotations in Explorer, and anticipated features in Portal such as publishing. However, product
requirements have since changed, and the user
annotations feature is being disabled in CZI's deployment of the Explorer.  
With no other feature in Explorer requiring authentication, the authentication feature is also being
disabled.

This document has therefore been rewritten to reflect the current state.

## Problem Statement | Background

1. The scope of this document is to establish user identity; authorization is
   not part of this design.
1. This document assumes that Auth0/OIDC/OAuth will be the technologies used
   for authentication, alternative authentication services are not considered
   here.

## Product Requirements

Data Portal

1. The Portal client will provide an authenticated user with access
   to features that require authentication.
1. An authenticated user will have the option to logout.
1. An unauthenticated user will not have access to features that require
   authentication. They will be disabled in the app, and require login on first
   use.
1. An unauthenticated user will have an option to login
1. The Portal backend will enforce the authentication for routes that
   require authentication by checking for the ID token, and rejecting requests
   that lack a token or have an invalid/expired token. The signature needs to be
   validated to ensure that the token has not been tampered with. An expired ID
   token can be refreshed (as long as the refresh token has not expired).
1. The Data Portal backend will have an endpoint to check if the requestor
   is authenticated: (<api_base_url>/v1/userinfo)
1. The token will be automatically refreshed
   when the ID token expires. The
   expiration date of the Refresh token could be set to the maximum of 30 days.

General

1. The login experience requires the use of cookies. This is considered "strictly necessary
   cookies" and it does not require consent. According to the GDPR documentation:
   "While it is not required to obtain consent for these cookies,
   what they do and why they are necessary should be explained to the user." (<https://gdpr.eu/cookies/>)

## Detailed Design | Architecture | Implementation

### Authentication Approach

1. OAuth2: an authorization standard that allows a user to grant limited access to their
   resources on one site, to another site, without having to expose their credentials.
1. OpenID Connect (OIDC): an identity layer on top of OAuth2.
1. Json Web Tokens (JWT): a compact and self-contained way for securely transmitting
   information between parties as a JSON object. This will be used to store user identity information.
1. Auth0: A flexible drop in solution to add authentication and authorization services. Uses
   industry standards like OAuth2, OIDC and JWT. [https://auth0.com/docs/getting-started/overview
   ](https://auth0.com/docs/getting-started/overview)
1. Identity providers: Initial set includes the following in priority order: Google, Github,
   ORCID, Facebook, Email. We may provide a “contact us” link on the authentication landing page
   to let the user request another identity provider.

### Authentication in Data Portal

#### User Experience

When an unauthenticated user visits a dataset page:

1. a “Log In” button will be displayed near the top of the page.
1. When the user clicks the “Log In” button the authentication flow starts.

When an authenticated user visits a dataset page:

1. The user's name appears in the top right corner. Clicking the name displays the user's email
   address, and the option to click "Log Out".
1. Logged in users can click on "My Collections" to see their collections. From there they can
   create collections and do other actions.
1. When the user clicks “Log Out” button, the logout flow starts.

#### Login Flow

The flow is similar to “regular app” flow in the auth0 documentation. The “Web App” is the
Data Portal backend in the diagram.

![Authentication Code Flow](imgs/authcodeflow.png)

Tl;dr: The following section goes over the above diagram, as it relates to the Data Portal,
and contains additional steps about storing the id_token in an httpOnly cookie so that user
identity can be safely stored and sent back to the backend on each request.

The process: numbers in square brackets refer to steps in the above diagram.

- The login request. The client sends a request to the “<api_base_url>/v1/login” endpoint of the
  Data Portal backend. It may supply a redirect path to this request, which is the path where the
  browser will be redirected after login.
- **[1]** The Data Portal backend then initiates the authentication request by redirecting the
  browser to the Auth0 Login Page.
  This is a full page navigation to the Auth0 CZI
  science tenant. The image below shows what that looks like. The Data Portal backend has been
  configured with a callback url, client_id, domain, client secret, and possibly other
  parameters needed for authentication. It initiates the authentication request with these
  parameters. The state variable that includes the current url of the dataset is attached to
  the request.
- **[2]** Auth0 has redirected the user to an auth0 provided login page (which can be branded
  with Cellxgene logos). The user can authenticate with an identity provider (like Google or others).

![Authentication Code Flow](imgs/authlogin.png)

- **[3]** Once the user has authenticated, Auth0 will direct the browser to the callback url
  in the Data Portal backend. The callback url must be known to both Auth0 and to the data portal.
  The callback url is "<api_base_url>/v1/oauth2/callback". The request will contain the authorization code.
- **[4]** The Data Portal backend can then exchange the authorization code for an ID token by
  sending a back channel request to Auth0. The Data Portal backend must know the client_secret for
  this step. This is stored in the AWS Secret Manager.
- **[5]** Auth0 will send back the ID token, as a JWT (JSON Web Token).
- The Data Portal backend then redirects (full page nav) the user back to the specified
  redirect page, and directs the client to save the JWT in an httpOnly cookie on the client.
- The JWT proves the identity of the client.
  1. If the ID token is invalid, then the Data Portal backend can re-initiate the authentication
     process by going back to step [1]
  2. The Data Portal backend will verify the signature of the token to ensure that it was
     signed by the
     sender and not altered.
  3. If the ID token is expired, then the Data Portal backend will obtain a new ID token
     automatically
     using the Refresh token. It will then save the new ID token in the cookie.
- Subsequent requests from the client to the
  Data Portal backend will include the cookie with the id_token.

#### Logout flow

The Logout flow is much simpler.

- A request is sent to the to the “<api_base_url>/v1/logout” endpoint in the Data Portal backend.
- The Data Portal redirects back to the root page, with a directive to delete the cookie containing the JWT.

#### Application Endpoints

To support the Auth0 authentication, following endpoints are needed by Data Portal backend. These
endpoints
were referenced in the previous sections, and defined here in more detail.

- <api_base_url>/v1/login?redirect=\<redirect\> : this endpoint is used to initiate the login
  process.  
  The server then initiated the authentication
  request by redirecting the browser to the Auth0 Login Pager.  
  After successful login, the client will be redirected to the path provided in the redirect
  parameter.
  This is step 1 in the diagram.
- <api_base_url>/v1/logout : this endpoint is used to clear the authentication credentials. The
  cookie
  containing the ID token is removed.
- <api_base_url>/v1/oauth2/callback: this endpoint is accessed during step 3 of the diagram
  . Auth0 redirects
  the browser to this endpoint, and the request contains the authentication code.
  The Data Portal backend then exchanges the
  authentication code for an ID Token. Once complete, the Data Portal backend redirects the client
  back to the dataset, and directs the browser to store the ID Token in a cookie.
- <api_base_url>/v1/userinfo: Returns information about the user (if authenticated), such as id, name, and email.

### Single Sign On

One of the main goals of the original authentication design was to support single sign on.
Single sign on in this case means that a user can login into either Data Portal or Explorer, and
automatically be logged into the other. Or they can log out of one, and automatically be
logged out of the other.

While this use case is not currently needed, it is already implemented in both Data Portal and
Explorer.  
It is described here in the case that a need for authentication in the Explorer comes up again.

Single sign on is achieved by using an httpOnly sameSite cookie, using the same domain for both
backends, and using the same format to store the cookie's data. Both applications use the
same Auth0 client id and domain. Auth0 will be
configured to register a callback to each of the two applications. A cookie setup this way
will be sent to the backend of both Data Portal and Explorer during API requests. Both
applications will set this cookie on login, and remove it on logout.

For best security, both applications should use the same full domain name so that we may use the
“sameSite=strict” option on the cookie. For example: api.cellxgene.cziscience.com. This will
keep the two applications isolated. The API to each application can be distinguished using a
different directory within that domain:

Front-end:

1. Explorer: cellxgene.cziscience.com/&lt;dataroot>/&lt;datasetname>/...
1. Data Portal: cellxgene.cziscience.com/

Back-end:

1. Explorer: api.cellxgene.cziscience.com/cellxgene/...
1. Data Portal: api.cellxgene.cziscience.com/dp/

## References

[0][location to the authorization code flow diagram](https://auth0.com/docs/architecture-scenarios/web-app-sso/part-1)

[1][auth0 docs](https://auth0.com/docs/)

[2][authentication code flow with proof key for code exchange (pkce](https://auth0.com/docs/flows/concepts/auth-code-pkce))
