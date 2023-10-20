# URI

## Resource Path

An API's [resource path](/docs/naming-conventions?id=uri-components) consists of URI's path, query and fragment components. It would include API's major version followed by namespace, resource name and optionally one or more sub-resources. For example, consider the following URI.

`https://api.foo.com/v1/vault/credit-cards/CARD-7LT50814996943336KESEVWA`

Following table lists various pieces of the above URI's resource path.

| Path Piece                      | Description                          | Definition                                                                                                                                                                                              |
| ------------------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `v1`                            | Specifies major version 1 of the API | The API major version is used to distinguish between two backward-incompatible versions of the same API. The API major version is an integer value which MUST be included as part of the URI.           |
| `vault`                         | The namespace                        | Namespace identifiers are used to provide a context and scope for resources. They are determined by logical boundaries in the business capability model implemented by the API platform.                |
| `credit-cards`                  | The resource name                    | If the resource name represents a collection of resources, then the `GET` method on the resource should retrieve the list of resources. Query parameters should be used to specify the search criteria. |
| `CARD-7LT50814996943336KESEVWA` | The resource ID                      | To retrieve a particular resource out of the collection, a resource ID MUST be specified as part of the URI. Sub-resources are discussed below.                                                         |

#### Sub-Resources

Sub-resources represent a relationship from one resource to another. The sub-resource name provides a meaning for the relationship. If cardinality is 1:1, then no additional information is required. Otherwise, the sub-resource SHOULD provide a sub-resource ID for unique identification. If cardinality is 1:many, then all the sub-resources will be returned. No more than two levels of sub-resources SHOULD be supported.

| Example                                                                          | Description                                                                                                                                                                                                                                                                                                                                                       |
| -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET https://api.foo.com/v1/customer-support/disputes/ABCD1234/documents`        | This call should return all the documents associated with dispute ABCD1234.                                                                                                                                                                                                                                                                                       |
| `GET https://api.foo.com/v1/customer-support/disputes/ABCD1234/documents/102030` | This call should return only the details for a particular document associated with this dispute. Keep in mind that this is only an illustrative example to show how to use sub-resource IDs. In practice, two step invocations SHOULD be avoided. If second identifier is unique, top-level resource (e.g. `/v1/customer-support/documents/102030`) is preferred. |
| `GET https://api.foo.com/v1/customer-support/disputes/ABCD1234/transactions`     | The following example should return all the transactions associated with this dispute and their details, so there SHOULD NOT be a need to specify a particular transaction ID. If specific transaction ID retrieval is needed, `/v1/customer-support/transactions/ABCD1234` is preferred (assuming IDs are unique).                                               |

### Resource Identifiers

[Resource identifiers](/docs/interpeting-guidelines?id=resource-identifier) identify a resource or a sub-resource. These **MUST** conform to the following guidelines.

- The lifecycle of a resource identifier MUST be owned by the resource's domain model, where they can be guaranteed to uniquely identify a single resource.
- APIs MUST NOT use the database sequence number as the resource identifier.
- A [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier), Hashed Id ([HMAC](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code) based) is preferred as a resource identifier.
- For security and data integrity reasons all sub-resource IDs MUST be scoped within the parent resource only.<br />
  **Example:** `/users/1234/linked-accounts/ABCD`<br /> Even if account "ABCD" exists, it MUST NOT be returned unless it is linked to user: 1234.
- Enumeration values can be used as sub-resource IDs. String representation of the enumeration value SHOULD be used.
- There MUST NOT be two resource identifiers one after the other.<br/>
  **Example:** `https://api.foo.com/v1/payments/payments/12345/102030`
- Resource IDs SHOULD try to use either Resource Identifier Characters or ASCII characters. There **SHOULD NOT** be any ID using UTF-8 characters.
- Resource IDs and query parameter values MUST perform URI percent-encoding for any character other than URI unreserved characters. Query parameter values using UTF-8 characters MUST be encoded.

## Query Parameters

Query parameters are name/value pairs specified after the resource path, as prescribed in RFC 3986.
[Naming Conventions](/docs/naming-conventions.md) should also be followed when applying the following section.

#### Filter a resource collection

- Query parameters SHOULD be used only for the purpose of restricting the resource collection or as search or filtering criteria.
- The resource identifier in a collection SHOULD NOT be used to filter collection results, resource identifier should be in the URI.
- Parameters for pagination SHOULD follow [pagination](#pagination) guidelines.
- Default sort order SHOULD be considered as undefined and non-deterministic. If a explicit sort order is desired, the query parameter `sort` SHOULD be used with the following general syntax: `{field_name}|{asc|desc},{field_name}|{asc|desc}`. For instance: `/accounts?sort=date_of_birth|asc,zip_code|desc`

#### Query parameters on a single resource

In typical cases where one resource is utilized (e.g. `/v1/payments/billing-plans/P-94458432VR012762KRWBZEUA`), query parameters SHOULD NOT be used.

#### Cache-friendly APIs

In rare cases where a resource needs to be highly cacheable (usually data with minimal change), query parameters MAY be utilized as opposed to `POST` + request body. As `POST` would make the response uncacheable, `GET` is preferred in these situations. This is the only scenario in which query parameters MAY be required.

#### Query parameters with POST

When `POST` is utilized for an operation, query parameters are usually NOT RECOMMENDED in favor of request body fields. In cases where `POST` provides paged results (typically in complex search APIs where `GET` is not appropriate), query parameters MAY be used in order to provide hypermedia links to the next page of results.

#### Passing multiple values for the same query parameter

When using query parameters for search functionality, it is often necessary to pass multiple values. For instance, it might be the case that a resource could have many states, such as `OPEN`, `CLOSED`, and `INVALID`. What if an API client wants to find all items that are either `CLOSED` or `INVALID`?

- It is RECOMMENDED that APIs implement this functionality by repeating the query parameter. This is inherently supported by HTTP standards and already built in to most client libraries.

  - The query above would be implemented as `?status=CLOSED&status=INVALID`.
  - The parameter MUST be marked as repeatable in API specifications using `"repeated": true` in the parameter's definition section.
  - The parameter's name SHOULD be singular.

- URIs have practical length limits that are quite low - most conservatively, about [2,000 characters](https://stackoverflow.com/questions/417142/what-is-the-maximum-length-of-a-url-in-different-browsers#417184). Therefore, there are situations where API designers MAY choose to use a single query parameter that accepts comma-separated values in order to accommodate more values in the query-string. Keep in mind that server and client libraries don't consistently provide this functionality, which means that implementers will need to write additional string parsing code. Due to the additional complexity and divergence from HTTP standards, this solution is NOT RECOMMENDED unless justified.
  - The query above would be implemented as `?statuses=CLOSED,INVALID`.
  - The parameter MUST NOT be marked as repeatable in API specifications.
  - The parameter MUST be marked as `"type": "string"` in API specifications in order to accommodate comma-separated values. Any other `type` value MUST NOT be used. The parameter description should indicate that comma separated values are accepted.
  - The query-parameter name SHOULD be plural, to provide a hint that this pattern is being employed.
  - The comma character (Unicode `U+002C`) SHOULD be used as the separator between values.
  - The API documentation MUST define how to escape the separator character, if necessary.