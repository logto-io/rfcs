---
Start date: 2023-09-27
Author(s): gao
Revision: 1
---

# Management API scopes

## Overview

In Logto, machine-to-machine applications can utilize the Management API to programmatically access Logto resources. This proposal suggests the addition of scopes to the Management API to enhance access control.

## Motivation

Currently, the Management API offers only the `all` scope, granting complete access. This isn't ideal from both security and usability perspectives.

For instance, a user might want to provide a third-party application with read-only access to logs without permitting log creation or deletion.

The introduction of scope sets can also facilitate Logto Cloud's collaboration feature, which lays the groundwork for various roles in the future.

## Detailed explanation

### Scope composition

Each scope will adhere to the naming convention of `action:resource`. This division is easily comprehensible to users and suits RESTful API design.

We define the following actions as the initial set:

- `read`: for reading resources
- `write`: for creating or updating resources
- `delete`: for deleting resources

Here's a table listing resources and their associated actions:

| Resource      | Read | Write | Delete | Description                                                                          |
| ------------- | ---- | ----- | ------ | ------------------------------------------------------------------------------------ |
| dashboard     | Y    | N     | N      | Statistics for new, active, and total users.                                         |
| apps          | Y    | Y     | Y      | All apps in the tenant.                                                              |
| resources     | Y    | Y     | Y      | All API resources and their permissions in the tenant except for the Management API. |
| experience    | Y    | Y     | N      | Sign-in experience settings, including custom phrases for the tenant.                |
| mfa           | Y    | Y     | N      | Multi-factor authentication settings for the tenant.                                 |
| connectors    | Y    | Y     | Y      | Email, SMS, and social connectors and their factories in the tenant.                 |
| users         | Y    | Y     | Y      | All users in the tenant.                                                             |
| logs          | Y    | Y     | Y      | All logs in the tenant, including user, app, and hook logs.                          |
| roles         | Y    | Y     | Y      | All roles in the tenant.                                                             |
| hooks         | Y    | Y     | Y      | All hooks in the tenant.                                                             |
| domains       | Y    | Y     | Y      | Custom domains for the tenant.                                                       |
| verifications | N    | Y     | N      | Verification sessions for the tenant.                                                |
| assets        | Y    | Y     | N      | All assets, such as branding logos, in the tenant.                                   |

The table essentially lists all supported scopes. For example, the `dashboard` resource has only one scope, `read:dashboard`, whereas the `apps` resource has three scopes: `read:apps`, `write:apps`, and `delete:apps`.

The complete scope set can be updated over time. The creator of the changes SHOULD update this specification accordingly.

### Deprecation of `all` scope

The `all` scope is overly broad and SHOULD be deprecated. However, to ensure backward compatibility, it SHOULD be retained for at least six months after the new scope set is introduced.

Here's the plan for implementing this breaking change:

1. [Day 0] Introduce the new scope set to the Management API, announce the beta release, and mention the planned deprecation of the `all` scope.
2. [Day 30] Remove the beta label for the new scope set, and add deprecation notices or labels (indicating that the `all` scope will be removed in six months) in the following places:
  - Management API documentation
  - Logto Console
  - Newsletter
3. [Day 90] Send a deprecation email to all users who have used the `all` scope in the past 90 days.
4. [Day 150] Repeat step 3.
5. [Day 180] Repeat step 3 and remove the `all` scope from the Management API documentation and Logto Console.
6. [Day 210] Remove the `all` scope from the Management API.

### Scope coverage

The new scope set SHOULD encompass all resources in the Management API. If the `scope` field is absent or empty in the access token, the Management API SHOULD return an error.

## Drawbacks

- The new scope set is not backward compatible with the current `all` scope, causing breaking changes and additional work for users during migration.
- The new scope set requires periodic updates, adding to the workload for both the team and users to keep up with changes.
  - Logto may consider providing an API to list all scopes in the future to assist users in staying up to date with changes.

## Rationale and alternatives

Several alternatives were considered:

- Introducing new scopes while retaining the `all` scope.
  - This approach leaves security concerns unaddressed and violates the principle of minimal scope.
- Introducing new scopes with the format of `resource:action`.
  - This approach deviates from RESTful API design and may lead to confusion when adding sub-resources in the future.

The current proposal offers several advantages:

- It defines access control in a more granular manner and allows for the extension of sub-resources. For example, the `logs` resource can be extended to include `logs:app` and `logs:user` in the future.
- It adheres to the minimal scope principle, promoting security best practices.
- It can be utilized for Logto Cloud's collaboration feature.

Not implementing these changes could result in the following issues:

- The `all` scope is overly permissive and inadequate for machine-to-machine applications, especially those involving third-party applications.
- Logto Cloud's collaboration feature would lack standardized scopes.

## Implementation

### Add new scopes to the Management API resource

The Management API resource SHOULD be updated to incorporate the new scope set. Before implementing organization, this also applies to the Management API resources in the admin tenant.

### Update Logto core to support the new scopes

The existing global authorization middleware for protected resources can be retained for recognizing identity and the `all` scope. A new authorization middleware in Logto Core that recognizes the new scopes should be added.

The authorization middleware SHOULD be put at least at the router level, rather than globally. To enforce the inclusion of scopes for all protected resources, the middleware MAY require a scope parameter in the constructor. The original global middleware MAY generate an error if no scope is enforced after the `next()` Promise is resolved.

### Update the admin access role

The admin access role SHOULD be updated to include all the new scopes and remove the `all` scope.

## Unresolved questions

- How should other collaboration roles be established in Logto Cloud? This pertains more to product decisions and can be addressed with the product team.
- When implementing the collaboration feature, admin users will directly access the Management API in the Console by organization.
  - Do we need to synchronize the scope set with the Management API?
  - If so, how can this synchronization be maintained?

## Future possibilities

- Further divide resources into more granular sub-resources and introduce additional scopes if necessary.
- Support dynamic scopes for specific resources. For example, the `app` resource could have a dynamic scope like `read:app:<app_id>`, representing read access to a specific app.
