---
Start date: '2023-09-25'
Author(s): [gao]
---

# Organization

**Table of contents**
- [1. Summary](#1-summary)
- [2. Motivation](#2-motivation)
- [3. Introduction](#3-introduction)
  - [3.1 Terminology](#31-terminology)
  - [3.2 Limitations](#32-limitations)
- [4. Organization template](#4-organization-template)
  - [4.1. Definition](#41-definition)
  - [4.2. Organization permission](#42-organization-permission)
  - [4.3. Organization role](#43-organization-role)
  - [4.4. Assign roles to organization members](#44-assign-roles-to-organization-members)
  - [4.5. Role-based access control](#45-role-based-access-control)
  - [4.6. Relationship to existing specs](#46-relationship-to-existing-specs)
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
- [8. Rationale and alternatives](#8-rationale-and-alternatives)
- [9. Future possibilities](#9-future-possibilities)

## 1. Summary

Organization is a new concept that gathers multiple identities together (mostly users). While Logto leverages [OpenID Connect](https://openid.net/specs/openid-connect-core-1_0.html) for authentication, it doesn't define the concept of organization. This spec proposes a foundation for organization support built on top of OpenID Connect.

## 2. Motivation

The product design of organization is locked down, but the current implementation and framework have no definition or support for it. This spec aims to define the basic concepts and requirements for organization from the engineering perspective, which will be used as the foundation for the implementation.

## 3. Introduction

> Before we start, it's worth noting that "organization" is different from "group". Group is usually used to group multiple identities together for batch operations, while organization also has its own roles and permissions (scopes) as defined in [Section 4](#4-organization-template).

### 3.1 Terminology

- **Organization**: An entity that may hold multiple permissions and resources, and link to multiple identities.
- **Membership**: A link between an identity and an organization.
- **Organization member**: An identity that has a membership with the organization.
- **Organization permission**: An action that can be granted or performed within the context of an organization.
  - In the following sections, we may use the term "scope" to compromise with the OpenID Connect terminology. The word "scope" and "permission" are interchangeable in this spec.
- **Organization role**: A set of permissions that can be granted to an organization member.
- **Organization template**: A set of organization permissions and roles that applies to every organization.

### 3.2 Limitations

In this spec, we only consider users as the identity type for simplicity. The support for other identity types (e.g. service accounts) can be discussed in the future.

## 4. Organization template

For modern SaaS applications, it's common to create multiple organizations with the same set of permissions and roles (the template). For example, a user may have the access to multiple workspaces in Slack, and the access to each workspace can vary. In this case, roles and permissions need to be specified in the context of an organization.

Although, duplicating the template for each organization would be doable for this use case, it will take notable effort for updating the template. Consider adding a new role named "collaborator", we need to perform the same interaction on every organization. The consistency across all organizations is required.

### 4.1. Definition

An organization template doesn't not belong to any organization. It is a tenant-level resource that consists of organization permissions and roles.

Each organization role in the template SHOULD only be granted with organization permissions in the same template. An organization role MAY have no permission granted.

> In this spec, we only discuss the sole organization template situation. A tenant that holds multiple organization templates is out-of-scope.

### 4.2. Organization permission

An organization permission should be a string with a meaningful value, and it also represents the name and unique identifier in the organization template. Permission names should be close to human-readable. For instance, `read:books` indicates the action to read or fetch books in the system, and `update:books` indicates the action to update book information.

It is RECOMMENDED to only use lowercase letters with certain symbols (`:_`) for permissions.

### 4.3. Organization role

An organization role has an unique identifier and holds a set of organization permissions. Each permission the role holds MUST be a predefined permission in the organization template.

### 4.4. Assign roles to organization members

Before assigning roles to an identity, the identity MUST have a membership with the organization, i.e. the identity is an organization member.

An organization member can be assigned with multiple roles in the organization. Each role the member holds MUST be a predefined role in the organization template.

The full set of permissions that an organization member can be granted is the union of all permissions that the member's roles have been granted.

### 4.5. Role-based access control

To authorize an actor (identity) to perform a certain action in an organization context, the actor MUST be assigned with at least one role that holds the desired permission (scope).

If the actor does not specify the organization context, the authorization MUST fail.

Note that organization role is unconsidered in authorization. The actor is only authorized to perform the action based on the organization permission, regardless of the organization role.

### 4.6. Relationship to existing specs

Sometimes, organization permissions (scopes) may be compared with the `scope` in OpenID Connect and [Resource Indicators](https://www.rfc-editor.org/rfc/rfc8707.html#name-access-token-request). Since organization is aiming to solve different problems for authentication and authorization, organization scopes and roles have no relationship to these existing specs, other than the sharing names ("scope" and "role"), and should be treated separately.

## 5. Authentication

In OpenID Connect, [ID token](https://openid.net/specs/openid-connect-core-1_0.html#IDToken) is a JWT token that contains the identity information of the user. In addition to the [standard claims](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims), we add a few new claims to the ID token to support organization.

### 5.1 Organization claims

| Name               | Type                  | Description                                                                                                                                                                                          |
| ------------------ | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| organization_ids   | array of strings      | The unique organization identifiers that the user has a membership with.                                                                                                                             |
| organization_roles | array of JSON objects | An array of organization roles that the user has. Each object contains the following fields: <ul><li>`organization_id`: The unique organization identifier in string.</li><li>`roles`: The role names that the user has been granted in the organization.</li></ul> |

The objects of `organization_roles` array should follow the following rules:

- The `organization_id` field should be unique in the array.
- The `roles` field should be an array of strings, and each string should be unique in the array.
- The `roles` field should not be empty.

These rules can save the space of the ID token and be enforced by the OP (OpenID Provider) when issuing the ID token.

An example of the ID token with organization claims:

```json
{
  "sub": "12345",
  "name": "John Wick",
  "organization_ids": ["12345", "67890", "13579"],
  "organization_roles": [
    {
      "organization_id": "12345",
      "roles": ["admin"]
    },
    {
      "organization_id": "67890",
      "roles": ["viewer", "editor"]
    }
  ]
}
```

### 5.2 Requesting claims using scope values

Based on the existing authentication flow, the OP (OpenID Provider) should be able to accept the following additional `scope` value in the authentication request, and provide the corresponding claim in the ID token:

- `organization`: The organization membership information of the user. This scope value requests access to the `organization_ids` and `organization_roles` claims.

## 6. Authorization

We leverage the same token endpoint defined in OAuth 2.0 and add a new grant type for requesting JWT access tokens in the context of an organization.

The JWT enforcement can enable the offline authorization for the resource server, which can reduce the network latency and improve the performance.

### 6.1. Token request

OAuth 2.0 defines the [token endpoint](https://www.rfc-editor.org/rfc/rfc6749.html#section-3.2) for requesting access tokens. Although it is doable to add more parameters to the existing grants (e.g. [RFC 8707](https://www.rfc-editor.org/rfc/rfc8707.html#name-access-token-request) uses this approach),  it will be more clear to define a new grant type for requesting access tokens for an organization context.

> The biggest challenge of adding more parameters for this spec is that we need to change the inner OP implementation, but since we are creating an extension out of the traditional specs, there's no reason for the upstream to accept our changes. Thus we choose to define a new grant type.

#### 6.1.1 Token parameters

To request an access token for an organization context, the client MUST send a request to the token endpoint with the following parameters:

- `grant_type` (REQUIRED): The grant type. The value MUST be `urn:logto:grant-type:organization_token`.
- `refresh_token` (REQUIRED): The refresh token issued to the client. The refresh token is the same as defined in [OpenID Connect](https://openid.net/specs/openid-connect-core-1_0.html#RefreshTokens) and [RFC OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749.html#section-1.5), and it MUST have the `organization` scope.
- `organization_id` (REQUIRED): The unique identifier of the organization.
- `scope` (OPTIONAL): The scopes that the client is requesting. The value MUST be a space-delimited list of strings. If omitted, the authorization server SHOULD issue an access token with the scopes granted by the user.

#### 6.1.2 Successful token response

If the request is valid and the authorization server successfully issues an access token, the response body requirements are the same as [Section 12.2](https://openid.net/specs/openid-connect-core-1_0.html#RefreshTokenResponse) in OpenID Connect.

Additionally, the access token MUST be a JWT token that contains the following claims:

- `jti`: The unique identifier of the access token.
- `iss`: The issuer identifier of the authorization server.
- `sub`: The user identifier.
- `organization_id`: The unique identifier of the organization.
- `exp`: The expiration time of the access token.
- `iat`: The time when the access token was issued.
- `scope`: The scopes in a whitespace-delimited string, which MUST be a subset of the scopes granted by the user.

A non-normative example of the access token:

```json
{
  "jti": "abcde",
  "iss": "https://logto.app",
  "sub": "12345",
  "organization_id": "12345",
  "exp": 1632571200,
  "iat": 1632484800,
  "scope": "read:books update:books"
}
```

#### 6.1.3. Error response

If the request is invalid, the authorization server SHOULD return an error response with the same format as defined in [Section 5.2](https://www.rfc-editor.org/rfc/rfc6749.html#section-5.2) in OAuth 2.0.

Note that the authorization server SHOULD return the same error response (`invalid_request`) for both invalid organization ID and other issues, such as the user is not a member of the organization, to prevent the client from guessing the organization information.

### 6.2. Token validation

The resource server MUST validate the access token in the following manner before granting access to the specific organization:

1. The access token MUST be a JWT token.
2. The access token MUST have a valid signature according to the algorithm defined in the `alg` header parameter and the JWKS (JSON Web Key Set) of the OP (OpenID Provider).
3. The `iss` claim MUST exactly match the issuer identifier of the OP.
4. The `organization_id` claim MUST exactly match one of the valid organization identifiers in the resource server.
5. The `exp` claim MUST be greater than or equal to the current Unix time. The resource server MAY add a reasonable leeway to this time.
6. The `iat` claim MUST be less than or equal to the current Unix time.
7. The `scope` claim MUST be a subset of the permissions in the organization template.

## 7. Drawbacks

[To be filled]

Nothing is perfect. What are the downsides of this proposal?

## 8. Rationale and alternatives

[To be filled]

Describe at least 2 alternatives to this proposal and answer the following questions:

- Why this proposal is better than the alternatives?
- What's the trade-off?
- What's the impact of not doing this?

## 9. Future possibilities

[To be filled]

Is there anything that's out of scope for this proposal but could be done in the future? What are the evolution possibilities of this proposal?
