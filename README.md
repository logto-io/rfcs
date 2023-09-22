# Logto RFCs

Usually, there's no need to create a RFC for changes since they can be properly handled by the normal code review process. However, some changes may be considered as "RFC-worthy" if:

- They are big enough to require a discussion before implementation.
- They introduce a new feature or a new way of doing things rather than leveraging existing implementations or open standards.
- They affect the overall architecture of the project.

The judgement of whether a change is "RFC-worthy" is left to the discretion of the project maintainers.

## RFC lifecycle

1. Create a new pull request with the RFC document in the `draft` directory.
2. Merge the pull request after the RFC is approved.
3. Implement the RFC.
4. Create a new pull request that moves the RFC document to the `active` directory.
