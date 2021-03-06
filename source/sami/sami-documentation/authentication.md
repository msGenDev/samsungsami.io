---
title: "Authentication"

---
# Authentication

SAMI relies on OAuth2 to authenticate users. If you aren't familiar with OAuth2 and its authentication flow, the documentation and libraries at the [official homepage](http://oauth.net/2/) will make your life easier.

A user or application must first obtain an access token from SAMI to make API calls. OAuth2 offers several methods of obtaining access tokens, and we support the following:

-   [Authorization Code:](/sami/sami-documentation/authentication.html#authorization-code-method) The user is directed to a UI that allows him
    to sign in or create an account. A successful signin or signup
    results in a redirection to your application's server with an
    authorization code. Your application must then exchange that code on
    the server side for an access token that can directly be used with
    our API calls. This method is used for when it is possible to keep
    the application secret secure, such as on the server side of a web
    application.
-   [Implicit:](/sami/sami-documentation/authentication.html#implicit-method) The user is directed to a UI that allows him to sign in
    or create an account. A successful signin or signup results in a
    redirection to your application's server with a URL fragment
    directly containing an access token. The access token can then
    directly be used in our API calls. This method is less secure, and
    is suitable for applications that cannot protect the application
    secret enough, such as mobile applications (the code of the
    application can be easily reverse-engineered).
-   [Client Credentials:](/sami/sami-documentation/authentication.html#client-credentials-method) This allows applications to get access tokens
    so they can run API calls as an application rather than as an
    authenticated user. There is no sign-in/sign-up or user interface
    involved. The access token can then directly be used in our API
    calls that accept this kind of token. This method is suitable for server-side code, as it requires the application secret to be passed. Before accessing user data in this flow, the user must have granted your application permissions on his data during either an [Authorization code](#authorization-code-method) or [Implicit](/sami/sami-documentation/authentication.html#implicit-method) flow.

All of our API calls need to be authenticated either as a user or as an
application. The authentication is done by obtaining an access token
from our accounts server and passing that access token back to the API
calls, either as an HTTP Authorization header (preferred method), or as
a URL parameter for WebSocket calls, as these normally do not support HTTP
headers.

For examples of how to build OAuth2 flows in your applications, see [**OAuth2 flow examples**](/sami/sami-documentation/oauth2-flow-examples.html).
{:.info}

## Get an application ID

In order to obtain access tokens, you will need an application ID and
the associated information. [Learn more about requesting an application ID.](http://developer.samsungsami.io/sami/sami-documentation/developer-user-portals.html#how-to-find-your-application-id)

## Obtain a token

### Authorization Code method

The Authorization Code method is done in multiple legs. First, your application must redirect the user to our accounts server with a call to `authorize`.

    GET /authorize

**Example**

    https://accounts.samsungsami.io/authorize?client_id=9628eef2a00d43d89b757b8d34373588&response_type=code&redirect_uri=https://myapp.com/callback&state=abcdefgh&scope=read,write

**Request parameters**

  |Parameter         |Description
  |----------------- |------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  |`client_id`{:.param}       |Your application ID (obtained in the first step).
  |`response_type`{:.param}  |The response expected. Must be `code` in this case, as we are requesting an authorization code.
  |`redirect_uri`{:.param}   |(Optional) The redirect URI you supplied when requesting the application ID. This is the URL that will be called back and passed the authorization code.
  |`state`{:.param}          |(Optional) A value (must be URL-safe) that is passed back to you when the flow is over. This is useful to keep a state and can be whatever you wish.
  |`scope`{:.param}          |(Optional) The type of permissions the application is requesting over the user's data, as a comma-separated list. For example: `read`. If omitted its default value is `read,write`.

The user is sent to Samsung Accounts, where she may sign in or create a new account. If the login is successful, the user will be asked to grant specific permissions on her data.

When the user clicks "Grant", she will be redirected to your server (at the `redirect_uri`) with an authorization code.

**Example**

    https://myapp.com/callback?code=0ee7fcd0abed470182b02cd649ec1c98&state=abcdefgh

The code can be exchanged for an access token within 60 seconds before expiring. For security, this should be done server-side. Among the request parameters below, `client_id` and `client_secret` should be included in an HTTP Authorization header, and the remaining parameters should be passed in the POST body. Consult [Sending client ID and client secret](#sending-clientid-and-clientsecret) for details on how to include them in an HTTP POST request.

**Request parameters**

  |Parameter         |Description
  |----------------- |----------------------------------------------------------------------------------------------------------------
  |`client_id`{:.param}      |Your application ID.
  |`client_secret`{:.param}  |Your application secret.
  |`grant_type`{:.param}     |The type of token being requested. In this case, we are requesting an Authorization Code type of access token.
  |`redirect_uri`{:.param}   |(Optional) The redirect URI you supplied to us for your application.
  |`code`{:.param}           |The authorization code returned by the authorize call.

**Example request** 

~~~
POST /token HTTP/1.1
Host: accounts.samsungsami.io
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https://myapp.com/callback&state=abcdefgh&scope=read,write
~~~

**Example response**

~~~~json
{
    "access_token":"2YotnFZFEjr1zCsicMWpAA",
    "token_type":"bearer",
    "expires_in":3600,
    "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA"
}
~~~~

**Response parameters**

  |Parameter         |Description
  |----------------- |-----------------------------------------------------------------------------------------------------------------------
  |`access_token`{:.param}   |The final access token that can be used with API calls.
  |`token_type`{:.param}     |Always `bearer` for now.
  |`expires_in`{:.param}     |In seconds, indicates how long the access token will expire.
  |`refresh_token`{:.param}  |A refresh token is used to obtain short-lived tokens. 

### Implicit method

In situations where `client_secret` cannot be kept safe with the application—for example, because the application code can easily be decompiled and reverse-engineered (mobile apps, desktop apps)—an Implicit access code grant is useful. 

This is less secure than the authorization code method, as it does not require making use of `client_secret` to obtain the code. However, it is more secure for the user than having the application directly request the user's name and password.

	GET /authorize

**Example**

    https://accounts.samsungsami.io/authorize?client_id=9628eef2a00d43d89b757b8d34373588&response_type=token&redirect_uri=https://myapp.com/callback&state=abcdefgh&scope=read,write

**Request parameters**

  |Parameter         |Description
  |----------------- |-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  |`client_id`{:.param}      |Your application ID.
  |`response_type`{:.param}  |The response expected. Must be `token` in this case, as we are requesting an access token.
  |`redirect_uri`{:.param}   |(Optional) The redirect URI you supplied to us for your application. This is the URL that will be in the redirection response.
  |`state`{:.param}          |(Optional) A value (must be URL-safe) that is passed back to you when the flow is over. This is useful to keep a state and can be whatever you wish.
  |`scope`{:.param}          |(Optional) The type of permissions the application is requesting over the user's data, as a comma-separated list. For example: `read`. If omitted its default value is `read,write`.

The user is sent to SAMI Accounts, where she may sign in or create a new account. If the login is successful, the user will be asked to grant specific permissions on her data.

When the user clicks "Grant", your application will receive a `302` (redirection) response in which the access token can be parsed from the URL fragment.

**Example response**

~~~~ 
HTTP/1.1 302 Found
Location: http://example.com/cb#access_token=2YotnFZFEjr1zCsicMWpAA&state=xyz&token_type=bearer&expires_in=7200
~~~~

**Response parameters**

  |Parameter        |Description
  |---------------- |-----------------------------------------------------------------
  |`access_token`{:.param}  |The final access token that can be used with API calls.
  |`token_type`{:.param}    |Always `bearer` for now.
  |`expires_in`{:.param}    |In seconds, indicates in how long the access token will expire.

### Client Credentials method

This type of access token allows an application to authenticate itself in situations where no user is directly involved. However, before the application can use this method to access a user's data, it should have asked the user to grant such access via UI, using the [Authorization Code method](#authorization-code-method) or the [Implicit method.](#implicit method)

This is an HTTP POST call that must be done server-side because it requires passing `client_secret`. Among the request parameters below, `client_id` and `client_secret` should be included in an HTTP Authorization header, and the remaining parameters should be passed in the POST body. Consult [Sending client ID and client secret](#sending-clientid-and-clientsecret) for details on how to include them in an HTTP POST request.

**Request parameters**

  |Parameter         |Description
  |----------------- |-----------------------------------------------------------------------------------------------------------------
  |`client_id`{:.param}      |Your application ID.
  |`client_secret`{:.param}  |Your application secret.
  |`grant_type`{:.param}     |The type of token being requested. In this case, we are requesting a Client Credentials type of access token.
  |`scope`{:.param}          |(Optional) The type of permissions the application is requesting over the user's data, as a comma-separated list. For example: `read`. If omitted its default value is `read,write`.

**Example request**

~~~ 
POST /token HTTP/1.1
Host: accounts.samsungsami.io
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&scope=read,write
~~~

**Example response**

~~~~
{
    "access_token":"2YotnFZFEjr1zCsicMWpAA",
    "token_type":"bearer",
    "expires_in":3600
}
~~~~

**Response parameters**

  |Parameter        |Description
  |---------------- |-----------------------------------------------------------------
  |`access_token`{:.param}  |The final access token that can be used with API calls.
  |`token_type`{:.param}    |Always `bearer` for now.
  |`expires_in`{:.param}    |In seconds, indicates in how long the access token will expire.

### Refresh a token

If an application obtains an access token using the [Authorization Code method](#authorization-code-method), it can refresh the token after the token expires. The application makes a POST call at the server side using a previously issued `refresh_token`. Among the request parameters below, `client_id` and `client_secret` should be included in an HTTP Authorization header, and the remaining parameters should be passed in the POST body. Consult [Sending client ID and client secret](#sending-clientid-and-clientsecret) for details on how to include them in an HTTP POST request.

**Request parameters**

|Parameter |Description
|---------|----------
|`grant_type`{:.param} |Must be set to value `refresh_token`.
|`refresh_token`{:.param} |The refresh token issued to the client.

**Example request**

~~~
POST /token HTTP/1.1
Host: accounts.samsungsami.io
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
~~~

**Example response**

~~~json
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache
{
  "access_token":"2YotnFZFEjr1zCsicMWpAA",
  "token_type":"bearer",
  "expires_in":3600,
  "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
  "scope": "the_scope"
}
~~~

### Sending client_id and client_secret

Sending `client_id` and `client_secret` is necessary when:

- Obtaining an access token using the [Authorization Code method.](#authorization-code-method)
- Obtaining an access token using the [Client Credentials method.](#client-credentials-method)
- [Refreshing an access token.](#refresh-a-token) 

HTTP Basic authentication is the recommended way. You can pass an Authorization header with Base64-encoded `client_id` and `client_secret` as follows:

**Example**

~~~
Authorization: Basic czZCaGRSa3F0Mzo3RmpmcDBaQnIxS3REUmJuZlZkbUl3
~~~

Alternatively, the credentials (`client_id` and `client_secret`) can be included in the request body (**not** the request URI). This should only be used when HTTP Basic authentication is not possible. 

**Request parameters**

 |Parameter |Description
  |---------|----------
  |`client_id`{:.param}      |Your application ID.
  |`client_secret`{:.param}  |Your application secret.

In the following example, an HTTP request to refresh an access token includes `client_id` and `client_secret` in the request body, instead of in the Authorization header. Please note that this is **not** a recommended method of sending `client_id` and `client_secret`. Use HTTP Basic authentication whenever possible.

**Example**

~~~
     POST /token HTTP/1.1
     Host: server.example.com
     Content-Type: application/x-www-form-urlencoded

     grant_type=refresh_token&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
     &client_id=s6BhdRkqt3&client_secret=7Fjfp0ZBr1KtDRbnfVdmIw
~~~

### /authorize errors

For authorization code grants, the error code is
included as an "error" query string parameter. For implicit grants, the error code is included as an "error" fragment parameter.

  |Http Status   |Error message                                                              |Condition                                                                                                                                                                         
  |------------- |---------------------------------------------------------------------- |--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ----------------------------------------------------
  |`400`{:.param}        |Invalid parameter: redirect_uri                                       |The provided redirect_uri does not match the registered redirect_uri for the client application.                                                                                
  |`302`{:.param}        |server_error                                                          |Unexpected server error                                                                                                                                                           Sent to "redirect_uri", includes the state param.
  |`500`{:.param}        |There was a server error without a valid redirect_uri to resend it.   |An unexpected server error and invalid redirect_uri was provided.                                                                                                                
  |`302`{:.param}        |invalid_request                                                       |The request is missing a required parameter (other than redirect_uri), includes an unsupported parameter value, includes a parameter more than once or is otherwise malformed.   Sent to "redirect_uri", includes the state param.
  |`302`{:.param}        |unauthorized_client                                                   |The client is not authorized to request an authorization code using this method.                                                                                                  Sent to "redirect_uri", includes the state param.
  |`302`{:.param}        |access_denied                                                         |The user or the authorization server denied the request.                                                                                                                          Sent to "redirect_uri", includes the state param.

### /token errors

The error is returned as a JSON object in a response like this:

~~~~
HTTP/1.1 400 Bad Request
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
   "error":"invalid_request"
}
~~~~

  |Http Status   |Error message                      |Condition                                                                                                                                                                           |
  |------------- |-------------------------- |---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |-------
  |`400`{:.param}        |server_error              |Unexpected server error.                                                                                                                                                           |
  |`400`{:.param}        |invalid_request           |The request is missing a required parameter (other than `redirect_uri`), includes an unsupported parameter value, includes a parameter more than once, or is otherwise malformed.  |
  |`400`{:.param}        |unauthorized_client       |The client is not authorized to request an authorization code using this method.                                                                                                   |
  |`302`{:.param}        |unsupported_grant_type   |The authorization grant type is not supported by the authorization server.                                                                                                         |

### Logout

In order to correctly start the logout flow, you need to call this service, so it can invalidate the SAMI session, and then give you the
control to invalidate your application session.

When finished, the user will be redirected to the given `redirect_uri` with the parameter `status=logout`.

	GET /logout

**Example**

    https://accounts.samsungsami.io/logout?redirect_uri=https://myapp.com/callback&state=abcdefgh

**Request parameters**

  |Parameter        |Description
  |---------------- |------------------------------------------------------------------------------------------------------------------------------------------------------
  |`redirect_uri`{:.param}  |The exact redirect URI you supplied to us for your application.
  |`state`{:.param}         |(Optional) A value (must be URL-safe) that is passed back to you when the flow is over. This is useful to keep a state and can be whatever you wish.

**Example response**

~~~~
HTTP/1.1 302 Found
Location: http://myapp.com/callback?status=logout&state=abcdefgh
~~~~

### Revoke a token

In order to completely invalidate the user sessions, you need to revoke the assigned tokens for them. This is a call that must be done server-side, because it requires passing the client credentials.

	PUT /revokeAccessToken

**Example**

    https://accounts.samsungsami.io/revokeAccessToken?client_credentials=9628eef2a00d43d89b757b8d34373588&token=0ea24090297b4108ae1338c39f25c118  

**Request parameters**

  |Parameter              |Description
  |---------------------- |---------------------------------------------------------------------------------
  |`client_credentials`{:.param}  |Your application credentials, obtained with a client credentials OAuth2 method.
  |`token`{:.param}               |The user token that you want to invalidate.

**Example response**

~~~~
{
  "data" : {
    "message" : "Token successfully revoked"
  }
}
~~~~

## Use an access token

### Pass the token as an HTTP header

This is the preferred way of authenticating your call. Pass the Authorization header as follows on your HTTP call:

    Authorization: bearer <access token>

**Example**

    Authorization: bearer 1c20060d9b9f4ad09ee16919a45c71b7

### Pass the token as a URL parameter

Only use this on WebSocket calls. ([Learn more about implementing WebSockets in SAMI for live-streaming data.](/sami/sami-documentation/sending-and-receiving-data.html#live-streaming-data-with-websocket-api)) 

Pass the Authorization header as a URL parameter:

    ?Authorization=bearer+<access token>

**Example**

    wss://api.samsungsami.io/v1.1/websocket?Authorization=bearer+1c20060d9b9f4ad09ee16919a45c71b7

