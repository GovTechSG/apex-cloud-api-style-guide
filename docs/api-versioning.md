# API Versioning

This section describes how to version APIs. It describes API's lifecycle states, enumerates versioning policy, describes backwards compatiblity related guidelines and describes an End-Of-Life policy.

## API Lifecycle

Following is an example of states of an API's lifecycle.

| State      | Description                                                                                                                                                                                                                 |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PLANNED    | API has been scheduled for development. API release plans have been established.                                                                                                                                            |
| BETA       | API is operational and is available to selected new subscribers in production for the purposes of testing, validating, and rolling out a new API.                                                                           |
| LIVE       | API is operational and is available to new subscribers in production. API version is fully supported.                                                                                                                       |
| DEPRECATED | API is operational and available at runtime to existing subscribers. API version is fully supported, including bug fixes addressed in backwards compatible way. API version is not available to new subscribers.            |
| RETIRED    | API is unpublished from production and no longer available at runtime to any subscribers. The footprint of all deployed applications entering this state must be completely removed from production and stage environments. |

## API Versioning Policy

API’s are versioned products and MUST adhere to the following versioning principles.

1. API specifications MUST follow the versioning scheme where where the `v` introduces the version, the major is an ordinal starting with `1` for the first LIVE release, and minor is an ordinal starting with `0` for the first minor release of any major release.
2. Every time there is an incremental change to an API, whether or not backward compatible, the API specification MUST be versioned. This allows the change to be labeled, documented, reviewed, discussed, published and communicated.
3. API endpoints MUST only reflect the major version.
4. API specification versions reflect interface changes and MAY be separate from service implementation versioning schemes.
5. A minor API version MUST maintain backward compatibility with all previous minor versions, within the same major version.
6. A major API version MAY maintain backward compatibility with a previous major version.

For a given functionality set, there MUST be only one API version in the `LIVE` state at any given time across all major and minor versions. This ensures that subscribers always understand which versioned API product they should be using. For example, v1.2 `RETIRED`, v1.3 `DEPRECATED`, or v2.0 `LIVE`.

## Backwards Compatibility

APIs SHOULD be designed in a forward and extensible way to maintain compatibility and avoid duplication of resources, functionality and excessive versioning.

APIs MUST adhere to the following principles to be considered backwards compatible:

1. All changes MUST be additive.
2. All changes MUST be optional.
3. Semantics MUST NOT change.
4. Query-parameters and request body parameters MUST be unordered.
5. Additional functionality for an existing resource MUST be implemented either:
   1. As an optional extension, or
   2. As an operation on a new child resource, or
   3. By altering a request body, while still accepting all the previous, existing request variations, if an existing operation (e.g., resource creation) cannot be reasonably extended.

### Non-exhaustive List of Backwards Incompatible Changes

<b>URIs</b>

1. A resource URI MAY support additional query parameters but they CANNOT be mandatory.
2. There MUST NOT be any change in the behavior of the API for request URIs without the newly added query parameters.
3. A new parameter with a required constraint SHALL NOT be added to a request.
4. The semantics of an existing parameter, entire representation, or resource SHOULD NOT be changed.
5. A service MUST recognize a previously valid value for a parameter and SHOULD NOT throw an error when used.
6. There MUST NOT be any change in the HTTP status codes returned by the URIs.
7. There MUST NOT be any change in the HTTP verbs (e.g. `GET`, `POST`, `PUT` or `PATCH`) supported earlier by the URI. The URI MAY however support a new HTTP verb.
8. There MUST NOT be any change in the name and type of the request or response headers of an URI. Additional headers MAY be added, provided they’re optional.

<b>APIs only support media type `application/json`. The following applies for JSON representation stability.</b>

1. An existing property in a JSON object of an API response MUST continue to be returned with same name and JSON type (number, integer, string, array, object).
2. If the value of a response field is an array, then the type of the contents in the array MUST NOT change.
3. If the value of the response field is an object, then the compatibility policy MUST apply to the JSON object as a whole.
4. New properties MAY be added to a representation any time, but it SHOULD NOT alter the meaning of an existing property.
5. New properties that are added MUST NOT be mandatory.
6. Mandatory fields are guaranteed to be present in the response.
7. For primitive types, unless there is a constraint described in the API documentation (e.g. length of the string, possible values for an ENUM type), clients MUST not assume that the values are constrained in any way.
8. If the property of an object is a URI, then it MUST have the same stability mentioned as URIs.
9. For an API returning HATEOAS links as part of the representation, the values of rel and href MUST remain the same.
10. For ENUM types, there MUST NOT be any change in already supported enumerated values or meaning of these values.

## End of Life Policy

The End-of-Life (EOL) policy regulates how API versions move from the `LIVE` to the `RETIRED` state. It is designed to ensure a consistent and reasonable transition period for API customers who need to migrate from the old to the new API version while enabling a healthy process to retire technical debt.

<b>Minor API Version EOL</b>

Per versioning policy, minor API versions MUST be backwards compatible with preceding minor versions within the same major version. Thus, minor API versions are `RETIRED` immediately after a newer minor version of the same major version becomes `LIVE`. This change should have no impact on existing subscribers so there is no need to transition through a `DEPRECATED` state to facilitate client migration.

<b>Major API Version EOL</b>

Per versioning policy, major API versions MAY be backwards compatible with preceding major versions. As such, the following rules apply when retiring a major API version.

1. A major API version MUST NOT be in the `DEPRECATED` state until a replacement service is `LIVE` that provides a clear customer migration path for all functionality that will be retained moving forward. This SHOULD include documentation and, as appropriate, migration tools and sample code that provide customers what they need to make a clean migration.
2. The deprecated API version MUST be in the `DEPRECATED` state for a minimum period of time to give client customers adequate notice to migrate. Deprecation of API versions with external clients SHOULD be considered on a case-by-case basis and may require additional deprecation time and/or constraints to minimize impact to the business.
3. If a versioned API in `LIVE` or `DEPRECATED` state has no clients, it MAY move to the `RETIRED` state immediately.

### End of Life Policy – Replacement Major API Version Introduction

Since a new major API version that results in deprecation of a pre-existing API version is a significant business investment decision, API owners MUST justify the new major version before beginning significant design and development work. API owners SHOULD explore all possible alternatives to introducing a new major API version with the objective of minimizing the impact on customers before deciding to introduce a new version. Justification SHOULD include the following:

Business Case

1. Customer value delivered by new version that is not possible with existing version.
2. Cost-benefit analysis of deprecated version versus the new version.
3. Explanation of alternatives to introducing an new major version and why those were not chosen.
4. If a backwards incompatible change is required to address a critical security issue, items 1 and 2 (above) are not required.

API Design

1. A domain model of all resources in the new API version and how they compare to the domain model of the previous major API version.
2. Description of APIs operations and use cases they implement.
3. Definition of service level objectives for performance and availability that are equal or better to the major API version to be deprecated.

Migration Strategy

1. Number of existing customers impacted; internal, external, and partners.
2. Communication and support plan to inform existing customers of new version, value, and migration path.