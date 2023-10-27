# Enterprise Cross-App Authorization

## Overview

Enterprises often have hundreds of SaaS applications.  SaaS applications often have integrations to other SaaS applications that are critical to the application experience and jobs to be done.  When a SaaS app needs to request an access token on-behalf of a user to a 3rd party SaaS integration's API, the end-user needs to complete an interactive delegated OAuth 2.0 ceremony and consent.  The SaaS application is not in the same security or policy domain as the 3rd party SaaS integration.

It is industry best practice for an enterprise to connect their ecosystem of SaaS applications to their Identity Provider (IdP) to centralize identity and access management capabilites for the organization.  End-users get a better experience (SSO) and administrators get better security outcomes such multi-factor authentication and zero-trust.  SaaS applications today enable the administrator to establish trust with an IdP for user authentication but typically don't allow the administrator to trust the IdP for API authorization.  

The draft specification [Authorization Cross Domain Code 1.0](https://openid.bitbucket.io/draft-acdc-01.html) (ACDC) defines a new JWT-based grant type that can requested from an Authorization Server and exchanged with another Authorization Server for Access and Refresh tokens.  This new grant enables federation for Authorization Servers across policy or administrative boundaries. The same enterprise IdP for example that is trusted by applications for SSO can be extended to broker access to APIs.  This enables the enteprise to centralize more access decisions across their SaaS ecosystem and provides better end-user experience for users that need to connect multiple applications via OAuth 2.0.

This specification extends support for the Authorization Cross-Domain Code (ACDC) grant to [Token Exchange] (https://datatracker.ietf.org/doc/html/rfc8693) requests, enabling applications to request access to 3rd party applications using backchannel operations that don't interupt the end user's interactive application experience.  Its also useful for deployments where SSO is SAML based and not using OpenID Connect.

## Roles

Requesting Application (Client)
: Application that wants to obtain an OAuth 2.0 access token on behalf of a signed-in user to an external/3rd party application's API that is managed by the same enterprise IdP.

Resource Application (Resource)
: Application that provides an OAuth 2.0 Protected Resource that is used across and enterprise's SaaS ecosystem.

Identity Provider (IdP)
: Organization's Identity Provider that is trusted by a set of applications in an enteprise's app ecosystem for identity and access management.



## Cross-Application Backchannel Flow with IdP

The example flow is for an enterprise `acme`


| Role     | App URL | Tenant URL   | Description |
| -------- | -------- | -------- | ----------- |
| Requesting Application | `https://wiki.app` | `https://acme.wiki.app` | SaaS Wiki app that embeds content from best-of-breed SaaS apps |
| Resource Application   | `https://chat.app` | `https://acme.chat.app` | SaaS chat and communication app |
| Identity Provider      | `https//idp.cloud`   | `https://acme.idp.cloud` | Cloud Identity Provider |


1. User logs in to the Requesting Application via SSO with the Enterprise IdP (SAML or OIDC)
2. Requesting Application requests an Authorization Cross-Domain Code (ACDC) for the Resource Application from the IdP
3. Requesting Application exchange the Authorization Cross-Domain Code (ACDC) for an access token at the Resource Application's token endpoint



### Pre-Conditions

- Requesting Application has a registered OAuth 2.0 Client with the IdP Authorization Server
- Requesting Application has a registered OAuth 2.0 Client with the Resource Application
- Enterprise has established a trust relationship between their IdP and the Requesting Application for SSO and Cross-Application Authorization
- Enterprise has established a trust relationship between their IdP and the Resource Application for SSO and Cross-Application Authorization
- Enterprise has granted the Requesting Application to act on-behalf of users for the Resource Application with a set of scopes

### Constraints

Token Exchange *should* only be supported for confidential clients.  Public clients *must* redirect with an OAuth 2.0 Authorization Request.

### SSO user with Enterprise IdP (SAML or OIDC)

Requesting Application initiates an authentication request with the tenant's trusted Enterprise IdP using SAML or OIDC.  

The following is an example using SAML 2.0

```http
302 Redirect
Location: https://acme.idp.cloud/SAML2/SSO/Redirect?SAMLRequest={base64AuthnRequest}&RelayState=DyXvaJtZ1BqsURRC
```

The user authenticates with the IdP and post backs an assertion to the Requesting Application

> The Enterprise IdP may enforce security controls such as multi-factor authentication before granting the user access to the Requesting Application. 

```
POST /SAML2/SSO/ACS HTTP/1.1
Host: https://acme.wiki.app/SAML2/ACS
Content-Type: application/x-www-form-urlencoded
Content-Length: nnn
 
SAMLResponse={AuthnResponse}&RelayState=DyXvaJtZ1BqsURRC
```
#### SAML Assertion

```xml
<saml:Assertion
  xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
  ID="3711ecd6-7f94-45c7-81c5-4926254845f3"
  Version="2.0"
  IssueInstant="2023-06-05T09:20:05Z">
  <saml:Issuer>https://acme.idp.cloud</saml:Issuer>
  <ds:Signature
    xmlns:ds="http://www.w3.org/2000/09/xmldsig#">...</ds:Signature>
  <saml:Subject>
    <saml:NameID
      Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress">
      karl@acme.com
    </saml:NameID>
    <saml:SubjectConfirmation
      Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
      <saml:SubjectConfirmationData
        InResponseTo="6c9830e4-1332-4f49-ade0-f24f61196d7e"
        Recipient="https://acme.wiki.app/SAML2/ACS"
        NotOnOrAfter="2023-06-05T09:25:05Z"/>
    </saml:SubjectConfirmation>
  </saml:Subject>
  <saml:Conditions
    NotBefore="2023-06-05T09:15:05Z"
    NotOnOrAfter="2023-06-05T09:25:05Z">
    <saml:AudienceRestriction>
      <saml:Audience>https://acme.wiki.app</saml:Audience>
    </saml:AudienceRestriction>
  </saml:Conditions>
  <saml:AuthnStatement
    AuthnInstant="2023-06-05T09:20:00Z"
    SessionIndex="3711ecd6-7f94-45c7-81c5-4926254845f3">
    <saml:AuthnContext>
      <saml:AuthnContextClassRef>
        urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
     </saml:AuthnContextClassRef>
    </saml:AuthnContext>
  </saml:AuthnStatement>
</saml:Assertion>
```  

### Request Authorization Cross-Domain Code (ACDC) for Resource Application from the IdP

Requesting Application makes a [RFC 8693 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693) request to the IdP's Token Endpoint

* `requested_token_type=urn:ietf:params:oauth:grant-type:jwt-acdc` 
* `audience` - Specifies the Client ID of the Resource Application that the Requesting App wants to get access to (The Client ID is as registered by the Resource Application at the Requesting Application). Note: The IdP will need to store a mapping between its own Client ID of the Resource Application and the Client ID that the Requesting Application uses at the Resource Application.
* `scope` - The space-separated list of scopes at the Resource Application to include in the token
* `subject_token` - The SSO assertion (SAML or OpenID Connect ID Token) for the target end-user
* `subject_token_type` - For SAML2 Assertion: `urn:ietf:params:oauth:token-type:saml2`, or OpenID Connect ID Token: `urn:ietf:params:oauth:token-type:id_token`
* Provides client authentication to the IdP (the example below uses the more secure `private_key_jwt` method)

```http
POST /oauth2/token HTTP/1.1
Host: acme.idp.cloud
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&requested_token_type=urn:ietf:params:oauth:grant-type:jwt-acdc 
&audience=https://acme.chat.app
&scope=chat.read+chat.history
&subject_token=PHNhbWw6QXNzZXJ0aW9uCiAgeG1sbnM6c2FtbD0idXJuOm9hc2lzOm5hbWVzOnRjOlNBTUw6Mi4wOmFzc2VydGlvbiIKICBJRD0iaWRlbnRpZmllcl8zIgogIFZlcnNpb249IjIuMCIKICBJc3N1ZUluc3RhbnQ9IjIwMjMtMDYtMDVUMDk6MjA6MDVaIj4KICA8c2FtbDpJc3N1ZXI-aHR0cHM6Ly9hY21lLmlkcC5jbG91ZDwvc2FtbDpJc3N1ZXI-CiAgPGRzOlNpZ25hdHVyZQogICAgeG1sbnM6ZHM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvMDkveG1sZHNpZyMiPi4uLjwvZHM6U2lnbmF0dXJlPgogIDxzYW1sOlN1YmplY3Q-CiAgICA8c2FtbDpOYW1lSUQKICAgICAgRm9ybWF0PSJ1cm46b2FzaXM6bmFtZXM6dGM6U0FNTDoxLjE6bmFtZWlkLWZvcm1hdDplbWFpbEFkZHJlc3MiPgogICAgICBrYXJsQGFjbWUuY29tCiAgICA8L3NhbWw6TmFtZUlEPgogICAgPHNhbWw6U3ViamVjdENvbmZpcm1hdGlvbgogICAgICBNZXRob2Q9InVybjpvYXNpczpuYW1lczp0YzpTQU1MOjIuMDpjbTpiZWFyZXIiPgogICAgICA8c2FtbDpTdWJqZWN0Q29uZmlybWF0aW9uRGF0YQogICAgICAgIEluUmVzcG9uc2VUbz0iNmM5ODMwZTQtMTMzMi00ZjQ5LWFkZTAtZjI0ZjYxMTk2ZDdlIgogICAgICAgIFJlY2lwaWVudD0iaHR0cHM6Ly9hY21lLndpa2kuYXBwL1NBTUwyL0FDUyIKICAgICAgICBOb3RPbk9yQWZ0ZXI9IjIwMjMtMDYtMDVUMDk6MjU6MDVaIi8-CiAgICA8L3NhbWw6U3ViamVjdENvbmZpcm1hdGlvbj4KICA8L3NhbWw6U3ViamVjdD4KICA8c2FtbDpDb25kaXRpb25zCiAgICBOb3RCZWZvcmU9IjIwMjMtMDYtMDVUMDk6MTU6MDVaIgogICAgTm90T25PckFmdGVyPSIyMDIzLTA2LTA1VDA5OjI1OjA1WiI-CiAgICA8c2FtbDpBdWRpZW5jZVJlc3RyaWN0aW9uPgogICAgICA8c2FtbDpBdWRpZW5jZT5odHRwczovL2FjbWUud2lraS5hcHA8L3NhbWw6QXVkaWVuY2U-CiAgICA8L3NhbWw6QXVkaWVuY2VSZXN0cmljdGlvbj4KICA8L3NhbWw6Q29uZGl0aW9ucz4KICA8c2FtbDpBdXRoblN0YXRlbWVudAogICAgQXV0aG5JbnN0YW50PSIyMDIzLTA2LTA1VDA5OjIwOjAwWiIKICAgIFNlc3Npb25JbmRleD0iMzcxMWVjZDYtN2Y5NC00NWM3LTgxYzUtNDkyNjI1NDg0NWYzIj4KICAgIDxzYW1sOkF1dGhuQ29udGV4dD4KICAgICAgPHNhbWw6QXV0aG5Db250ZXh0Q2xhc3NSZWY-CiAgICAgICAgdXJuOm9hc2lzOm5hbWVzOnRjOlNBTUw6Mi4wOmFjOmNsYXNzZXM6UGFzc3dvcmRQcm90ZWN0ZWRUcmFuc3BvcnQKICAgICA8L3NhbWw6QXV0aG5Db250ZXh0Q2xhc3NSZWY-CiAgICA8L3NhbWw6QXV0aG5Db250ZXh0PgogIDwvc2FtbDpBdXRoblN0YXRlbWVudD4KPC9zYW1sOkFzc2VydGlvbj4
&subject_token_type=urn:ietf:params:oauth:token-type:saml2
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJhbGciOiJSUzI1NiIsImtpZCI6IjIyIn0.
```

The IdP evaluates administrator-defined policy for the token exchange request and determines if the application (client) should be granted access to act on behalf of the subject for the target audience & scopes.  

IdP may also introspect the authentication context described in the SSO assertion to determine if step-up authentication is required.

If access is granted, the IdP will return a signed Authorization Cross-Domain Code JWT

```
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "acdc": "eyJhbGciOiJIUzI1NiIsI...",
  "token_type": "urn:ietf:params:oauth:grant-type:jwt-acdc"
}
```

#### Authorization Cross-Domain Code JWT

The ACDC JWT is issued by the IdP `https://acme.idp.cloud` for the requested audience `https://acme.chat.app` and includes the following claims:

* `iss` - The IdP `issuer` URL
* `sub` - The User ID at the IdP
* `azp` - Client ID of the Requesting Application as registered with the Resource Application
* `aud` - Client ID of the Resource Application as registered with the IdP
* `exp` - 
* `iat` -
* `scopes` - Array of scopes at the Resource Application granted to the Requesting Application

```
{
  "iss": "https://acme.idp.cloud",
  "sub": "karl@acme.com",
  "azp": "https://acme.wiki.app",
  "aud": "https://acme.chat.app",
  "exp": 1311281970,
  "iat": 1311280970,
  "scopes" : [ "chat.read" , "chat.history" ]
}
```

#### Errors

On an error condition, the IdP returns an OAuth 2.0 Token Error response, e.g:

```
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{
  "error": "invalid_grant",
  "error_description": "Audience validation failed"
}
```



### Step-Up Authentication

The IdP may require step-up authentication for the subject if the authentication context in the subject's assertion does not meet policy requirements. An `insufficient_user_authentication` OAuth error response may be returned to convey the authentication requirements back to the client similar to [OAuth 2.0 Step-up Authentication Challenge Protocol](https://www.ietf.org/archive/id/draft-ietf-oauth-step-up-authn-challenge-17.html)

```
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{
  "error":"insufficient_user_authentication",
  "error_description":"Subject doesn't meet authentication requirements",
  "max_age: "5"
}
```
The Requesting Application would need to redirect the user back to the IdP to obtain a new assertion that meets the requirements and retry the token exchange.

> It may make more sense to request the ACDC as an additional `response_type` on the authorization request if using OIDC for SSO when performing a step-up to skip the need for additional token exchange round-trip


## Exchange Authorization Cross-Domain Code for 3rd Party SaaS API Access Token

The Requesting Application makes a request to the Resource Application's token endpoint, including the following parameters:

* `grant_type=urn:ietf:params:oauth:grant-type:jwt-acdc`
* `acdc` - The ACDC obtained in the previous step
* Client Authentication - the client authenticates with its credentials as registered with the Resource Application


```
POST /oauth2/token HTTP/1.1
Host: acme.chat.app
Authorization: Basic yZS1yYW5kb20tc2VjcmV0v3JOkF0XG5Qx2

grant_type=urn:ietf:params:oauth:grant-type:jwt-acdc
acdc=eyJhbGciOiJIUzI1NiIsI...
```

The Resource Application token endpoint responds with an OAuth 2.0 Token Response, e.g.:

```
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "token_type": "Bearer",
  "access_token": "2YotnFZFEjr1zCsicMWpAA",
  "expires_in": 86400,
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
}
```

## FAQ

### Why Profile JWT Bearer Assertion Grant?

TBD

[RFC7523](https://www.rfc-editor.org/rfc/rfc7523.html) defines the use of a JSON Web Token (JWT) Bearer Token as a means for requesting an OAuth 2.0 access token.  


