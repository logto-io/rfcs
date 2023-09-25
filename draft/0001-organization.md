---
Start date: 2023-09-25
Author(s): @gao-sun
---

**Table of contents**
- [Organization](#organization)
  - [1. Summary](#1-summary)
  - [2. Motivation](#2-motivation)
  - [3. Introduction](#3-introduction)
    - [3.1 Terminology](#31-terminology)
    - [3.2 Limitations](#32-limitations)
  - [4. ID token extension](#4-id-token-extension)
    - [4.1 Organization claims](#41-organization-claims)
    - [4.2 Requesting claims using scope values](#42-requesting-claims-using-scope-values)
  - [5. Role-based access control](#5-role-based-access-control)
  - [Drawbacks](#drawbacks)
  - [Rationale and alternatives](#rationale-and-alternatives)
  - [Implementation](#implementation)
  - [Future possibilities](#future-possibilities)

# Organization

## 1. Summary

Organization is a new concept that groups multiple identities together (mostly users). While Logto leverages [OpenID Connect](https://openid.net/specs/openid-connect-core-1_0.html) for authentication, it doesn't define the concept of organization. This spec proposes a foundation for organization support built on top of OpenID Connect.

## 2. Motivation

The product design of organization is locked down, but the current implementation and framework have no definition or support for it. This spec aims to define the basic concepts and requirements for organization from the engineering perspective, which will be used as the foundation for the implementation.

## 3. Introduction

> Before we start, it's worth noting that "organization" is different from "group". Group is usually used to group multiple identities together for batch operations, while organization also has its own roles and permissions (scopes).

### 3.1 Terminology

- **Organization**: An entity that may hold multiple permissions and resources, and link to multiple identities.
- **Membership**: A link between an identity and an organization.
- **Organization member**: An identity that has a membership with the organization.
- **Organization permission**: An action that can be granted or performed within the context of an organization.
  - In the following sections, we may use the term "scope" to compromise with the OpenID Connect terminology. The word "scope" and "permission" are interchangeable in this spec.
- **Organization role**: A set of permissions that can be granted to an organization member.

### 3.2 Limitations

In this spec, we only consider users as the identity type for simplicity. The support for other identity types (e.g. service accounts) can be discussed in the future.

## 4. ID token extension

In OpenID Connect, [ID token](https://openid.net/specs/openid-connect-core-1_0.html#IDToken) is a JWT token that contains the identity information of the user. In addition to the [standard claims](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims), we add a few new claims to the ID token to support organization.

### 4.1 Organization claims

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

### 4.2 Requesting claims using scope values

Based on the existing authentication flow, the OP (OpenID Provider) should be able to accept the following additional `scope` value in the authentication request, and provide the corresponding claim in the ID token:

- `organization`: The organization membership information of the user. This scope value requests access to the `organization_ids` and `organization_roles` claims.

## 5. Role-based access control

To be filled.

## Drawbacks

Nothing is perfect. What are the downsides of this proposal?

## Rationale and alternatives

Describe at least 2 alternatives to this proposal and answer the following questions:

- Why this proposal is better than the alternatives?
- What's the trade-off?
- What's the impact of not doing this?

## Implementation

Describe the steps and overview of the implementation. If you have any code, pseudo-code, or a proof of concept, please include it here.

## Future possibilities

Is there anything that's out of scope for this proposal but could be done in the future? What are the evolution possibilities of this proposal?
