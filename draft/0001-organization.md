---
Start date: 2023-09-25
Author(s): gao
Revision: 1
---

# Organization

**Table of contents**
- [1. Overview](#1-overview)
- [2. Motivation](#2-motivation)
- [3. Introduction](#3-introduction)
  - [3.1 Terminology](#31-terminology)
  - [3.2 Limitations](#32-limitations)
- [4. Organization](#4-organization)
  - [4.1. Definition](#41-definition)
  - [4.2. Organization template](#42-organization-template)
  - [4.3. Organization permission](#43-organization-permission)
  - [4.4. Organization role](#44-organization-role)
  - [4.5. Assign roles to organization members](#45-assign-roles-to-organization-members)
  - [4.6. Role-based access control](#46-role-based-access-control)
  - [4.7. Relationship to existing specifications](#47-relationship-to-existing-specifications)
- [5. Authentication](#5-authentication)
  - [5.1 Organization claims](#51-organization-claims)
  - [5.2 Requesting claims using scope values](#52-requesting-claims-using-scope-values)
- [6. Authorization](#6-authorization)
  - [6.1. Token request](#61-token-request)
    - [6.1.1 Token parameters](#611-token-parameters)
    - [6.1.2 Successful token response](#612-successful-token-response)
    - [6.1.3. Error response](#613-error-response)
  - [6.2. Token validation](#62-token-validation)
- [7. Drawbacks](#7-drawbacks)
  - [7.1. ID token size](#71-id-token-size)
  - [7.2. Allowed dynamic organizations](#72-allowed-dynamic-organizations)
  - [7.3. The absence of resource indicators](#73-the-absence-of-resource-indicators)
- [8. Rationale and alternatives](#8-rationale-and-alternatives)
- [9. Future possibilities](#9-future-possibilities)

## 1. Overview

Organization is a concept that brings together multiple identities (mostly users). While Logto leverages [OpenID Connect](https://openid.net/specs/openid-connect-core-1_0.html) for authentication, it lacks a formal framework for handling organizations. This specification aims to establish a foundation for organization support built on the OpenID Connect framework.

## 2. Motivation

The current Logto implementation lacks a clear definition and framework for supporting organizations. This specification seeks to define the fundamental concepts and requirements for organizations from an engineering perspective, providing the groundwork for its implementation.

## 3. Introduction

> Before we start, it's essential to distinguish between "organization" and "group". A group typically serves to group multiple identities for batch operations, while an organization has its own roles and permissions (scopes), as detailed in [Section 4](#4-organization-template).

### 3.1 Terminology

- **Organization**: An entity that can have multiple permissions and resources and is connected to multiple identities.
- **Membership**: A connection between an identity and an organization.
- **Organization member**: An identity with a membership in the organization.
- **Organization permission**: An action that can be granted or executed within the context of an organization.
  - Throughout this specification, the terms "scope" and "permission" are used interchangeably.
- **Organization role**: A set of permissions that can be assigned to an organization member.
- **Organization template**: A collection of organization permissions and roles that applies to every organization.

### 3.2 Limitations

For simplicity, this specification focuses on users as the identity type. Support for other identity types, such as service accounts, can be explored in the future.

## 4. Organization

### 4.1. Definition

An organization is an entity that can connect multiple identities. Each connection is called a membership, and the identity is called an organization member.

### 4.2. Organization template

In modern SaaS applications, it's common to create multiple organizations with the same set of permissions and roles (the template). For instance, a user may have access to multiple workspaces in Slack, each with varying levels of access. In such cases, roles and permissions must be defined within the context of an organization.

While duplicating the template for each organization is feasible, it can become challenging to update the template consistently. For example, adding a new role called "collaborator" would require repeating the same action for every organization. Maintaining consistency across all organizations is crucial.

An organization template is not associated with any specific organization; it is a resource at the tenant level that comprises organization permissions and roles.

> This specification focuses on a single organization template scenario. Multiple organization templates within a tenant are beyond the scope of this discussion.

### 4.3. Organization permission

An organization permission SHOULD be represented as a meaningful string, also serving as the name and unique identifier within the organization template. Permission names should be easily understandable. For example, `read:books` indicates the action to read or retrieve books in the system, while `update:books` signifies updating book information.

It is RECOMMENDED to only use lowercase letters with specific symbols (`:_`) for permissions.

### 4.4. Organization role

An organization role has a unique identifier and contains a set of organization permissions. Each organization role in the template SHOULD only be granted with predefined permissions in the same template. An organization role MAY have no permission granted.

### 4.5. Assign roles to organization members

Before assigning roles to an identity, the identity MUST have a membership with the organization, making them an organization member. An organization member can be assigned multiple roles within the organization, and each role held by the member must be predefined in the organization template. The complete set of permissions granted to an organization member is the union of all permissions granted by their roles.

### 4.6. Role-based access control

To authorize an actor (identity) to perform a specific action within an organization context, the actor MUST be assigned at least one role that contains the desired permission (scope).

If the actor does not specify the organization context, the authorization MUST fail.

It's important to note that organization roles are not considered in authorization. The actor is authorized to perform the action based solely on the organization permission, regardless of their organization roles.

### 4.7. Relationship to existing specifications

Organization permissions (scopes) are distinct from the `scope` in OpenID Connect and [Resource Indicators](https://www.rfc-editor.org/rfc/rfc8707.html#name-access-token-request). While these existing specifications share terminology like "scope" and "role" with organization, they are designed to address different authentication and authorization challenges. Organization scopes and roles should be treated separately.

## 5. Authentication

In OpenID Connect, the [ID Token](https://openid.net/specs/openid-connect-core-1_0.html#IDToken) is a JWT token containing user identity information. In addition to the [Standard Claims](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims), we introduce new claims to the ID token to support organization.

### 5.1 Organization claims

| Name          | Type                  | Description                                                                                                                                                                                                                                               |
| ------------- | --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| organizations | array of JSON objects | An array of organization ids and roles that the user has. Each object contains the following fields: <ul><li>`id`: The unique organization identifier as a string.</li><li>`roles`: The role names granted to the user within the organization.</li></ul> |

The objects in the `organizations` array must adhere to these rules:

- The `id` field must be unique within the array.
- The `roles` field must be an array of strings, with each string being unique within the `roles` array.
- The `roles` field must be defined. If the user has no roles within the organization, the `roles` field must be an empty array.

These rules help minimize the size of the ID token and can be enforced by the OpenID Provider (OP) when issuing the ID token.

An example of the ID token with organization claims:

```json
{
  "sub": "12345",
  "name": "John Wick",
  "organizations": [
    {
      "id": "12345",
      "roles": ["admin"]
    },
    {
      "id": "67890",
      "roles": ["viewer", "editor"]
    },
    {
      "id": "abcde",
      "roles": []
    }
  ]
}
```

### 5.2 Requesting claims using scope values

As per the existing authentication flow, the OpenID Provider (OP) SHOULD accept an additional `scope` value in the authentication request and provide the corresponding claim in the ID token:

- `urn:logto:scope:organization`: This scope value requests access to the `organizations` claim.

Here's an example of an authentication request with the `urn:logto:scope:organization` scope (values are for demonstration purposes only):

```
GET /authorize?
  response_type=code
  &scope=openid%20profile%20email%20urn:logto:scope:organization
  &client_id=s6BhdRkqt3
  &state=af0ifjsldkj
  &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb HTTP/1.1
Host: server.example.com
```

## 6. Authorization

We utilize the same Token Endpoint defined in OAuth 2.0 and introduce a new grant type for requesting JWT access tokens in the context of an organization.

Implementing JWT enforcement can enable offline authorization for the resource server, reducing network latency and improving performance.

To request scopes in the organization template, the client MUST add the exact scope values to the `scope` parameter in the authentication request. For example, if the organization template has the `read:books` permission, a non-normative authentication request may look like this (values are for demonstration purposes only):

```
GET /authorize?
  response_type=code
  &scope=openid%20profile%20email%20urn:logto:scope:organization%20read:books
  &client_id=s6BhdRkqt3
  &state=af0ifjsldkj
  &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb HTTP/1.1
Host: server.example.com
```

### 6.1. Token request

OAuth 2.0 defines the [Token Endpoint](https://www.rfc-editor.org/rfc/rfc6749.html#section-3.2) for requesting access tokens. Although it's possible to add more parameters to the existing grants (as [RFC 8707](https://www.rfc-editor.org/rfc/rfc8707.html#name-access-token-request) does), it's clearer to define a new grant type specifically for requesting access tokens within an organization context.

> The primary challenge of adding more parameters for this specification is the need to alter the inner OP implementation. Since we are creating an extension based on traditional specifications, there's no guarantee that upstream will accept our changes. Therefore, we opt to define a new grant type.

#### 6.1.1 Token parameters

To request an access token within an organization context, the client MUST send a request to the Token Endpoint with the following parameters:

- `grant_type` (REQUIRED): The grant type. The value MUST be `urn:logto:grant-type:organization_token`.
- `refresh_token` (REQUIRED): The refresh token issued to the client. This refresh token follows the same definition as in [OpenID Connect](https://openid.net/specs/openid-connect-core-1_0.html#RefreshTokens) and [RFC OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749.html#section-1.5), and it must have the `urn:logto:scope:organization` scope.
- `organization_id` (REQUIRED): The unique identifier of the organization. It MUST be an organization that the user has a membership with.
- `scope` (REQUIRED): The scopes that the client is requesting. The value MUST be a space-delimited list of strings. If a scope has not been granted to the user or it is not a valid scope in the organization template, the authorization server SHOULD ignore it.
  - The scope values MUST be a subset of the scopes granted to the user during authentication, and the authorization server MUST NOT grant any additional scopes.
  - If the `scope` parameter is not provided, the authorization server MUST return an error response with the `invalid_request` error code.

#### 6.1.2 Successful token response

If the request is valid, and the authorization server successfully issues an access token, the response body SHOULD meet the requirements outlined in [Section 12.2](https://openid.net/specs/openid-connect-core-1_0.html#RefreshTokenResponse) of OpenID Connect.

In addition, the access token MUST be a JWT token containing the following claims:

- `jti`: The unique identifier of the access token.
- `iss`: The issuer identifier of the authorization server.
- `sub`: The user identifier.
- `org`: The unique identifier of the organization.
- `exp`: The expiration time of the access token.
- `iat`: The time when the access token was issued.
- `scope`: The scopes, represented as a whitespace-delimited string, MUST be a subset of the scopes granted to the user.

A non-normative example of the access token:

```json
{
  "jti": "abcde",
  "iss": "https://logto.app",
  "sub": "12345",
  "org": "12345",
  "exp": 1632571200,
  "iat": 1632484800,
  "scope": "read:books update:books"
}
```

#### 6.1.3. Error response

If the request is invalid, the authorization server SHOULD return an error response in the same format as defined in [Section 5.2](https://www.rfc-editor.org/rfc/rfc6749.html#section-5.2) of OAuth 2.0.

Note that the authorization server SHOULD return the same error response (error code `invalid_request`) for both invalid organization ID and other issues, such as the user not being a member of the organization, to prevent the client from guessing organization information.

### 6.2. Token validation

Before granting access to the specific organization, the resource server MUST validate the access token as follows:

1. The access token MUST be a JWT token.
2. The access token MUST have a valid signature according to the algorithm defined in the `alg` header parameter and the JSON Web Key Set (JWKS) of the OpenID Provider (OP).
3. The `iss` claim MUST exactly match the issuer identifier of the OP.
4. The `org` claim MUST exactly match one of the valid organization identifiers in the resource server.
5. The `exp` claim MUST be greater than or equal to the current Unix time. The resource server MAY add a reasonable leeway to this time.
6. The `iat` claim MUST be less than or equal to the current Unix time.
7. The `scope` claim MUST be a subset of the permissions in the organization template.

## 7. Drawbacks

### 7.1. ID token size

The introduction of new claims may increase the size of the ID token. In a multi-tenant environment, the ID token size may significantly increase if the user has numerous memberships and roles.

To mitigate this issue, the OP MAY choose to include organization claims in the ID token only when the client requests the [Userinfo Endpoint](https://openid.net/specs/openid-connect-core-1_0.html#UserInfo) with the `urn:logto:scope:organization` scope.

### 7.2. Allowed dynamic organizations

Unlike Resource Indicators, which require the client to specify the resource identifier when making an authentication request, this specification allows the client to request access tokens for any organization the user is a member of. This is because resource servers typically don't interact directly with the user, but organization memberships are directly linked to the user.

In essence, organization memberships function more like an identity attribute. It is acceptable to request access tokens dynamically for any organization the user is a member of.

From a security standpoint, we have defined an organization template to restrict the organization scopes that the user can request. We also adhere to the OAuth 2.0 specification to ensure that organization scopes must be declared in the authentication request, and the authorization server must not grant any additional scopes for security reasons.
### 7.3. The absence of resource indicators

[Resource Indicators](https://www.rfc-editor.org/rfc/rfc8707.html) define a standardized way to request access tokens for specific resources, which is particularly useful for multiple API endpoints with varying permissions. However, applying this concept to organizations remains unclear and requires further discussion.

## 8. Rationale and alternatives

The concept of organizations is prevalent in modern SaaS applications, known by various names like workspaces, projects, and more, as seen in Slack, GitHub, and Google Workspace. Many of our users have expressed the need for this feature, making it a crucial addition to Logto.

During discussions, two key aspects emerged:

**Implementation**

Logto Cloud already follows a typical multi-tenant model for identity, and we initially used Resource Indicators to support it. However, we encountered two challenges:

- Duplicating permissions and roles for each tenant (organization) proved to be unscalable and difficult to maintain.
- There was no direct connection between an identity and the resources the identity has access to, requiring users to perform at least two authentication requests to obtain an access token for a specific tenant:
  1. Perform the an authentication request to fetch an access token for the Logto Cloud API resource.
  2. Read available tenants for the current user, and perform another authentication request to fetch an access token for the desired tenant.

**Role-based access control**

Some solutions directly use tenant roles and permissions for organizations, implying that permissions are tied to specific resources.

While this approach may be simpler to implement, it can confuse users and developers due to the blurred boundaries between organizations and resources.

In this specification, we introduce the organization template as a tenant-level resource, with organization permissions and roles exclusively defined within it. This approach clarifies the organization concept, decouples it from resources, and ensures that the organization feature can function independently without affecting existing resources.

## 9. Future possibilities

There are numerous possibilities for expanding the concept of organizations in the future, including:

- Support for other identity types, such as service accounts.
- Support for multiple organization templates within a tenant.
- Implementation of organization-level roles and permissions.
- Exploring resource indicators for organizations.
