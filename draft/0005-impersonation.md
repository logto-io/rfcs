---
Start date: 2024-05-30
Author(s): wangsijie
Revision: 1
---

# Impersonation

## 1. Overview

User impersonation is a feature that allows an authorized user to assume the identity of another user within the system. This capability can be useful for various reasons such as troubleshooting, providing support, and performing administrative tasks. This document aims to explore the design and implementation of user impersonation in Logto.

## 2. Motivation

As an enterprise-ready identity provider, it is essential to provide user impersonation capabilities to support various use cases. For example, a support engineer may need to troubleshoot an issue on behalf of a user, or an administrator may need to verify the permissions of a user. User impersonation can be helpful to these processes and improve the overall user experience.

## 3. Token exchange

The user impersonation feature will be implemented using the OAuth 2.0 [token exchange flow](https://www.rfc-editor.org/rfc/rfc8693.html). The token exchange flow allows an authorized user to exchange their access token for a new access token that represents the identity of another user. The new access token can then be used to make API requests on behalf of the impersonated user.

### 3.1 Request

The client makes a token exchange request to the tenant's token endpoint with a special grant type using the HTTP POST method. The following parameters are included in the HTTP request entity-body using the `application/x-www-form-urlencoded` format.

1. `grant_type`: REQUIRED. The value of this parameter must be `urn:ietf:params:oauth:grant-type:token-exchange` indicates that a token exchange is being performed.
2. `client_id`: REQUIRED. The client identifier of the client application that is making the request, the returned access token will contain this `client_id` claim.
3. `resource`: OPTIONAL. The resource indicator, the same as other token requests.
4. `scope`: OPTIONAL. The requested scopes, the same as other token requests.
5. `subject_token`: REQUIRED. A security token that represents the identity of the party on behalf of whom the request is being made. See the next section for more details.
6. `subject_token_type`: REQUIRED. The type of the security token provided in the `subject_token` parameter. The value of this parameter must be `urn:ietf:params:oauth:token-type:access_token`.
7. `actor_token`: OPTIONAL. A security token that represents the identity of the acting party, it should contain valid `accountId` and have `openid` scope. If present, the access token issued by Logto will contain an additional `act` claim that represents the identity of the acting party.
8. `actor_token_type`: OPTIONAL. The type of the security token provided in the `actor_token` parameter. The value of this parameter must be `urn:ietf:params:oauth:token-type:access_token`.

### 3.2 Response

If the token exchange request is successful, the tenant's token endpoint returns an access token that represents the identity of the impersonated user. The response includes the following parameters in the HTTP response entity-body using the `application/json` format.

1. `access_token`: REQUIRED. The access token that represents the identity of the impersonated user.
2. `issued_token_type`: REQUIRED. The type of the issued token. The value of this parameter must be `urn:ietf:params:oauth:token-type:access_token`.
3. `token_type`: REQUIRED. The type of the token. The value of this parameter must be `Bearer`.
4. `expires_in`: REQUIRED. The lifetime in seconds of the access token.
5. `scope`: OPTIONAL. The scopes of the access token.

### 3.3 Example token exchange

The following example demonstrates a token exchange request and response. An authorized user is exchanging the subject token for a new access token that represents the identity of another user. Here's a non-normative example of the token exchange request:

```
POST /oidc/token HTTP/1.1
Host: example.logto.app
Content-Type: application/x-www-form-urlencoded

client_id=mklwerdfw&grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&scope=openid%20offline_access%20profile
&subject_token=accVkjcJyb4BWCxGsndESCJQbdFMogUC5PbRDqceLTC
&subject_token_type=
urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aaccess_token
```

Logto validates the `subject_token` and other parameters in the token exchange request. If the request is valid, Logto generates a new access token that represents the identity of the impersonated user and returns it in the response. Here's a non-normative example of the token exchange response:

```
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-cache, no-store

{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjMiLCJzY29wZSI6Im9wZW5pZCBvZmZsaW5lX2FjY2VzcyBwcm9maWxlIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyLCJhY3QiOnsic3ViIjoiNDU2In19.H-X4W0mu-lJae27_rvsHAx5LpzsHTDYozkWDWzBrMoM",
  "scope": "openid offline_access profile",
  "issued_token_type": "urn:ietf:params:oauth:token-type:access_token",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

The payload of the access token:

```json
{
  "sub": "123",
  "scope": "openid offline_access profile",
  "client_id": "client1",
  "name": "John Doe",
  "iat": 1516239022
}
```

## 4. Subject token

The `subject_token` parameter in the token exchange request represents the identity of the party on behalf of whom the request is being made. The `subject_token` is a special one-time access token that is issued to the authorized user by Logto's Management API. The `subject_token` is an opaque token that is not intended to be used directly by the client application. Instead, the client application exchanges the `subject_token` for a new access token that represents the identity of the impersonated user. Because the `subject_token` is only consumed by Logto as the authorization server, it does not need to be a JWT.

### 4.1 Request

The client makes a request to the tenant's Management API to issue a subject token. The following parameters are included in the HTTP request entity-body using the `application/json` format.

1. `userId`: REQUIRED. The ID of the user on behalf of whom the request is being made.
2. `context`: OPTIONAL. Additional context information that may be used in "[Custom JWT claims](https://docs.logto.io/docs/recipes/custom-jwt/)". The context is an object that contains key-value pairs of additional claims.

### 4.2 Response

If the request to issue a subject token is successful, the tenant's Management API returns the subject token in the response.

1. `subjectToken`: REQUIRED. The subject token that represents the identity of the user.
2. `expiresIn`: REQUIRED. The lifetime in seconds of the subject token.

### 4.3 Example subject token issuance

The following example demonstrates a request to issue a subject token and the response. This request is made by a machine-to-machine application that has the necessary permissions to issue subject tokens.

```
POST /api/subject-tokens HTTP/1.1
Host: example.logto.app
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "userId": "123456",
  "context": {
    "ticketId": "7890"
  }
}
```

Logto validates the `userId` and permissions to issue a subject token. If the request is valid, Logto generates a new subject token that represents the identity of the user and returns it in the response. Here is an example of the response:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "subjectToken": "accVkjcJyb4BWCxGsndESCJQbdFMogUC5PbRDqceLTC",
  "expiresIn": 600
}
```

## 5. Additional claims in the access token

### 5.1 `act`

When `actor_token` presents in the request of token exchange, the access token issued by Logto will contain an additional `act` claim that represents the identity of the impersonated user. The `act` claim is an object that contains the following fields.

1. `sub`: REQUIRED. The identity of the acting user. Typically, this will be the party that is authorized to use the requested security token and act on behalf of the subject.

This is an example JWT access token with the `act` claim.

```json
{
  "aud": "https://resource.example.com",
  "iss": "https://example.logto.app",
  "exp": 1443904177,
  "sub": "user2",
  "act": {
    "sub": "user1"
  }
}
```

The `act` claim is used to indicate that `user1` is acting on behalf of `user2`.

## 6. Drawbacks

- The impersonation feature introduces a new token grant flow and requires careful implementation to ensure security and privacy.
- The new token exchange flow is not supported by Logto's existing libraries and SDKs, and additional client-side implementation is required.
- The subject token is one-time use, and the token exchange won't issue a refresh token, the client application needs to handle the expiration of the access token.

## 7. Rationale and alternatives

### 7.1 Alternatives

#### 7.1.1 Direct impersonation

In this alternative approach, the support engineer directly assumes the identity of the user without using the token exchange flow. The support engineer signs in as the user using the user's credentials or a special sign in flow without the need of the user's credentionals. This approach is simple, but it may lack the auditability and standard security controls.

#### 7.1.2 Issuing a custom JWT token

In this alternative approach, Logto provides an API that allow the support engineer to request a JWT token with custom claims, such as the user ID and permissions. The support engineer can use the custom JWT token to make API requests on behalf of the user. This approach is flexible, but it bypass the OIDC flow, and lose the benefits of validation among different parties and permissions.

### 7.2 Why this proposal is better than the alternatives?

- The token exchange flow provides a standardized way to implement user impersonation. And it is supported by the RFC8693 specification.
- The subject token allows Logto to control the issuance and usage of impersonation tokens, providing full auditability and security controls.
- This flow does not require UI interaction, which is more flexible.

### 7.3 What's the trade-off?

- The implementation of the token exchange flow requires additional development effort in the client-side.

### 7.4 What's the impact of not doing this?

- Without user impersonation, the developer will need to implement custom solutions to allow admin users to act on behalf of other users, which may introduce security risks and compliance issues.

## 8 Future possibilities

- Support direct sign in using subject token, which reduces the need for the client application to handle the impersonation flow.

## Appendix A. Example

The following example demonstrates how a support engineer can use the user impersonation feature to troubleshoot an issue on behalf of a user. Let's say the support engineer received a report from a user that they are unable to access a specific resource. The support engineer needs to investigate the issue and verify the user's permissions.

The following sequence diagram illustrates the flow of the user impersonation feature in Logto.

```mermaid
sequenceDiagram
    participant SE as Support engineer (frontend)
    participant RO as Resource owner API
    participant M2M as Logto Management API
    participant TS as Tenant's token endpoint
    participant RS as Resource

    SE ->> RO: Request for subject token
    RO ->> RO: Validate user permission
    RO ->> M2M: Request for subject token
    M2M ->> RO: Issue subject token
    RO ->> SE: Return subject token

    SE ->> TS: Request token endpoint with token exchange grant
    TS ->> SE: Issue access token

    SE ->> RS: Fetch user resource with the access token
    RS ->> SE: Return data
```

### Step 1: Issue a subject token

The support engineer calls the resource owner's API to issue a subject token for the user who reported the issue. The following request is a simple example of the resource owner's API. We assume the API is `POST /request-for-subject-token`, which requires the implementation of the necessary security controls to validate the support engineer's request according to their business requirements.

```
POST /request-for-subject-token HTTP/1.1
Host: resource.example.com
Authorization: Bearer ...
Content-Type: application/json

{
  "userId": "123456",
  "ticketId": "7890"
}
```

The resource owner API validates the support engineer's request and call Logto's Management API using a machine-to-machine application to issue a subject token for the user.

```
POST /api/subject-tokens HTTP/1.1
Host: example-tenant.logto.app
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "userId": "123456",
  "context": {
    "ticketId": "7890"
  }
}
```

Logto validates the request and issues a subject token for the machine-to-machine application:

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "subjectToken": "accVkjcJyb4BWCxGsndESCJQbdFMogUC5PbRDqceLTC",
  "expiresIn": 600
}
```

The resource owner API returns the subject token to the support engineer.

### Step 2: Exchange the subject token for an access token

The support engineer exchanges the subject token for an access token that represents the identity of the user in the frontend application:

```
POST /oidc/token HTTP/1.1
Host: example.logto.app
Content-Type: application/x-www-form-urlencoded

client_id=frontend-id&grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atoken-exchange
&scope=openid%20offline_access%20profile
&subject_token=accVkjcJyb4BWCxGsndESCJQbdFMogUC5PbRDqceLTC
&subject_token_type=
  urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aaccess_token
```

Logto validates the subject token and other parameters in the token exchange request. If the request is valid, Logto issues a new access token that represents the identity of the user and returns it in the response.

```
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-cache, no-store

{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjMiLCJzY29wZSI6Im9wZW5pZCBvZmZsaW5lX2FjY2VzcyBwcm9maWxlIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyLCJhY3QiOnsic3ViIjoiNDU2In19.H-X4W0mu-lJae27_rvsHAx5LpzsHTDYozkWDWzBrMoM",
  "scope": "openid offline_access profile",
  "issued_token_type": "urn:ietf:params:oauth:token-type:access_token",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

The support engineer receives the access token and can now make API requests on behalf of the user.

### Step 3: Investigate the issue

The support engineer uses the access token to make API requests to investigate the issue reported by the user. The support engineer can verify the user's permissions, troubleshoot the problem, and provide assistance to the user.

## Appendix B. Subject token context

The `context` parameter in the subject token issuance request is an optional field that allows the client application to provide additional context information that may be used in custom JWT claims. The `context` is an object that contains key-value pairs.

This will be stored in the subject token, and in the custom JWT function, the argument `context` will include a `grant` object that contains the subject token context.

The following is the `getCustomJwtClaims` function:

```ts
const getCustomJwtClaims = async ({ token, context, environmentVariables }) => {
  return {};
};
```

An example of the `context`:

```json
{
  "grant": {
    "type": "urn:ietf:params:oauth:grant-type:token-exchange",
    "subjectTokenContext": {
      "ticketId": "7890"
    }
  }
}
```
