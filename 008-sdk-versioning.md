## Revised SDK Versioning Strategy for Iroha Core

## Semantic Versioning

The SDKs will continue to follow [semantic versioning](https://semver.org/) with `MAJOR.MINOR.PATCH`.

- `MAJOR`: Incremented only when there are breaking changes in the SDK itself, regardless of Iroha Core's version.
- `MINOR`: Incremented for backward-compatible features or enhancements in the SDK.
- `PATCH`: Incremented for backward-compatible bug fixes or minor updates in the SDK.

## Relationship with Iroha Core

1. Initial Compatibility: Initially, set the SDK versions to align with Iroha Core `20.0.0`, reflecting the current release. For example, the SDKs might start at `2.0.0`.

2. Incremental SDK Updates: As we continue to deliver enhancements, features, and bug fixes for Iroha Core `2.0.0`, increment the SDK versions independently of Iroha Core's version. Use `MINOR` and `PATCH` version increments to indicate the level of changes in the SDKs themselves.

3. Documentation Updates: Continuously update the documentation to specify which version of Iroha Core your SDK versions are compatible with. This information should be prominently displayed for users to reference.

## Versioning Example

Let's say we have a Java SDK initially at version `2.0.0`, which aligns with Iroha Core `2.0.0`.

- If we make non-breaking enhancements to the Java SDK, we can increment the version to `2.1.0`.

- For bug fixes or minor updates that don't introduce breaking changes in the SDK, we can increment the version to `2.1.1`.

- Whenever you release updates, ensure that the documentation clearly states the compatibility of each SDK version with Iroha Core `2.0.0`.

### Continuous SDK Development

This strategy allows our SDKs to continue evolving and improving while keeping compatibility with Iroha Core `2.0.0`. By using `MINOR` and `PATCH` version increments, we can convey the level of changes within each SDK release. Make sure to maintain clear and up-to-date documentation to assist users in choosing the appropriate SDK version for their needs, considering compatibility with Iroha Core `2.0.0`.


