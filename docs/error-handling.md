# Error Handling

As per HTTP specifications, the outcome of a request execution could be specifiedusing an integer and a message. The number is known as the _status code_ and the message as the _reason phrase_. The reason phrase is a human readable message used to clarify the outcome of the response. The HTTP status codes in the `4xx` range indicate client-side errors (validation or logic errors), while those in the `5xx` range indicate server-side errors (usually defect or outage). However, these status codes and human readable reason phrase are not sufficient to convey enough information about an error in a machine-readable manner. To resolve an error, non-human consumers of RESTful APIs need additional help.

Therefore, APIs MUST return a JSON error representation that conforms to the [`error.json`][22] schema defined in the [Common Types][13] repository. It is recommended that the namespace that an API belongs to has an error catalog associated with it. Please refer to [Error Catalog](#error-catalog) for more details.

## Error Schema

An error response following `error.json` as schema MUST include the following fields:

- `name`: A human-readable, unique name for the error. It is recommended that this value would be retrieved from the error catalog [`error_spec.json#name`][24] before sending the error response.
- `details`: An array that contains individual instance(s) of the error with specifics such as the following. This field is required for client side errors (`4xx`).
  - `field`: [JSON Pointer][23] to the field in error if in body, else name of the path parameter or query parameter.
  - `value`: Value of the field in error.
  - `issue`: Reason for error. It is recommended that this value would be retrieved from the error catalog [`error_spec_issue.json#issue`][25] before sending the error response.
  - `location`: The location of the field in the error, either `query`, `path`, or `body`. If this field is not present, the default value is `body`.
- `debug_id`: A unique error identifier generated on the server-side and logged for correlation purposes.
- `message`: A human-readable message, describing the error. This message MUST be a description of the problem NOT a suggestion about how to fix it. It is recommended that this value would be retrieved from the error catalog [`error_spec.json#message`][24] before sending the error response.
- `links`: [HATEOAS](/docs/hypermedia.md) links specific to an error scenario. Use these links to provide more information about the error scenario and how to resolve it. You could insert links from [`error_spec.json#suggested_application_actions`][24] and/or [`error_spec.json#suggested_user_actions`][24] here as well as other HATEOAS links relevant to the API.

The following fields are optional:

- `information_link`: (`deprecated`) A URI for expanded developer information related to this error; this SHOULD be a link to the publicly available documentation for the type of error. Use `links` instead.

##### Use of JSON Pointer

If you have used some other means to identify the `field` in an already released API, you could continue using your existing approach. However, if you plan to migrate to the approach suggested, you would want to bump up the major version of your API and provide migration assistance to your clients as this could be a potential breaking change for them.

The JSON Pointer for the `field` SHOULD be a [JSON string value](https://tools.ietf.org/html/rfc6901#section-5).

### Input Validation Errors

In validating requests, there are a variety of concerns that should be addressed in the following order:

| Request Validation Issue                                                                                                                         | HTTP status code           |
| ------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------- |
| Not well-formed JSON.                                                                                                                            | `400 Bad Request`          |
| Contains validation errors that the client can change.                                                                                           | `400 Bad Request`          |
| Cannot be executed due to factors outside of the request body. The request was well-formed but was unable to be followed due to semantic errors. | `422 Unprocessable Entity` |

## Error Samples

This section provides some samples to describe usage of [`error.json`][22] in various scenarios.

#### Validation Error Response - Single Field

The following sample shows a validation error of type `VALIDATION_ERROR` in one field. Because this is a client error, a `400 Bad Request` HTTP status code should be returned.

```

{
   "name":"VALIDATION_ERROR",
   "details":[
      {
         "field":"#/credit_card/expire_month",
         "issue":"Required field is missing",
         "location":"body"
      }
   ],
   "debug_id":"123456789",
   "message":"Invalid data provided",
   "information_link":"http://developer.foo.com/apidoc/blah#VALIDATION_ERROR"
}
```

#### Validation Error Response - Multiple Fields

The following sample shows a validation error of the same type, `VALIDATION_ERROR`, in two fields. Note that `details` is an array listing all the instances in the error. Because both these are a client errors, a `400 Bad Request` HTTP status code should be returned.

```

{
   "name": "VALIDATION_ERROR",
   "details": [
      {
         "field": "/credit_card/expire_month",
         "issue": "Required field is missing",
         "location": "body"
      },
      {
         "field": "/credit_card/currency",
         "value": "XYZ",
         "issue": "Currency code is invalid",
         "location": "body"
      }
   ],
   "debug_id": "123456789",
   "message": "Invalid data provided",
   "information_link": "http://developer.foo.com/apidoc/blah#VALIDATION_ERROR"
}

```

#### Validation Error Response - Bulk

For heterogenous types of client-side errors shown below, `OBJECT_NOT_FOUND_ERROR` and `MULTIPLE_CORE_BUNDLES`, an array named `errors` is returned. Each error instance is represented as an item in this array. Because these are client validation errors, a `400 Bad Request` HTTP status code should be returned.

```

   "errors": [
      {
         "name": "OBJECT_NOT_FOUND_ERROR",
         "debug_id": "38cdd677a83a4",
         "message": "Bundle is not found.",
         "information_link": "<link to public doc describing OBJECT_NOT_FOUND_ERROR error>",
         "details": [
            {
               "field": "/bundles/0/bundle_id",
               "value": "33333",
               "issue": "BUNDLE_NOT_FOUND",
               "location": "body"
            }
         ]
      },
      {
         "name": "MULTIPLE_CORE_BUNDLES",
         "debug_id": "52cde38284sd3",
         "message": "Multiple CORE bundles.",
         "information_link": "<link to public doc describing MULTIPLE_CORE_BUNDLES error>",
         "details": [
            {
               "field": "/bundles/5/bundle_id",
               "value": "88888",
               "issue": "MULTIPLE_CORE_BUNDLES",
               "location": "body"
            }
         ]
      }
   ]
```

#### Semantic Validation Error Response 

In cases where client input is well-formed and valid but the request action may require interaction with APIs or processes outside of this URI, an HTTP status code `422 Unprocessable Entity` should be returned.

```
{
   "name": "BALANCE_ERROR",
   "debug_id": "123456789",
   "message": "The account balance is too low. Add balance to your account to proceed.",
   "information_link": "http://developer.foo.com/apidoc/blah#BALANCE_ERROR"
}
```

## Error Declaration In API Specification

It is important that documentation generation tools and client/server-side binding generation tools recognize [`error.json`][22]. Following section shows how you could refer `error.json` in an API specification confirming to OpenAPI.

```
"responses": {
          "200": {
            "description": "Address successfully found and returned.",
            "schema": {
              "$ref": "address.json"
            }
          },
          "403": {
            "description": "Unauthorized request.  This error will occur if the SecurityContext header is not provided or does not include a party_id."
          },
          "404": {
            "description": "The requested address does not exist.",
            "schema": {
              "$ref": "v1/schema/json/draft-04/error.json"
            }
          },
          "default": {
            "description": "Unexpected error response.",
            "schema": {
              "$ref": "v1/schema/json/draft-04/error.json"
            }
          }
        }

```

## Samples with Error Scenarios in Documentation

The User Guide of an API is a document that is exposed to API consumers. In addition to linking to samples showing successful execution for invocation on various methods exposed by the API, the API developer should also provide links to samples showing error scenarios. It is equally, or perhaps more, important to show the API consumers how an API would propagate errors in a machine-readable form in order to build applications that take necessary actions to handle errors gracefully and in a meaningful manner.

In conclusion, we reiterate the message we started with that non-human consumers of RESTful APIs need more help to take necessary actions to resolve an error in a machine-readable manner. Therefore, a representation of errors following the [schema](#error-schema) described here MUST be returned by APIs for any HTTP status code that falls into the ranges of 4xx and 5xx.

## Error Catalog

Error handling [guidelines](#error-handling) described earlier show how to provide error related details responses at runtime. This section explains how to catalog the errors so they can be easily consumed in service runtime and for generating documentation.

An Error Catalog is a single JSON file that contains a collection of error specifications (or error metadata) for a namespace. Each error specification includes error name, error message, issue details and related links among other things. The error catalog supports multiple locales or languages. For a specific error catalog, there should be exactly one default version, known as the top-level catalog, which could be in English for example. There should be corresponding locale-specific catalogs, one for each additional supported locale, as needed.

### Reasons To Catalog Errors

Following are some reasons to catalog the errors for an API:

1. To _externalize_ hard-coded error message strings from the API implementation: Developers are good at writing code but not necessarily good at writing error messages. Often error messages written by a developer make a lot of assumptions about the context and audience. It is hard to change such error messages if these are embedded in implementation code. For example, to change a message from "Add Card refused due to compliance guidelines" to `Could not add card due to failure to comply with guideline %s`, a service developer has to make the change and also redeploy the service emitting that error.
2. To _localize_ the error strings: If error related strings such as `message`, `issue`, `actions`, among other things in `error.json` are externalized, it is easy for the documentation and an internationalization team to modify and localize these without any help from service developers and without requiring redeployment of the services.
3. To keep service's implementation and the API documentation in _sync_ with regards to errors: API consumers should be able to refer with confidencethe API's documentation for errors generated by the services at runtime. This helps to reduce the cost and the time spent supporting an API and increases `adoptability` of the API.

### Error Catalog Schema

There are four main JSON schema files for the Error Catalog.

1. [`error_catalog.json`][26] defines top-level catalog container.

- `namespace`: API namespace
- `language`: language used to catalog the errors. Default is US English. This value MUST be a [BCP-47](https://tools.ietf.org/html/bcp47) language tag as in `en` or `en-US`.
- `errors`: one or more error catalog items.

2. [`error_catalog_item.json`][27] defines an item in error catalog. This schema in its initial version only includes error specification `error_spec`. In future versions, it would provide a space to establish relationship between an error specification and method specifications that would use the error specification to respond with error.
3. [`error_spec.json`][24] is where the core of error specification is defined. The specification includes the following properties.

- `name`: A human-readable, unique name for the error. This value MUST be the value set in [`error.json#name`][22] before sending the error response.
- `message`: A human-readable message, describing the error. This message MUST be a description of the problem NOT a suggestion about how to fix it. This value MUST be the value set in [`error.json#message`][22] before sending the error response. This value could be localized.
- `log_level`: Log level associated with this error. This MUST NOT be streamed out in error responses or exposed in any external documentation.
- `legacy_code`: Legacy error code. Use if and only if the existing and published error metadata uses the code and it must continue to be supported. Utilize `additionalProperties` of [`error.json`][22] to send this code in the error response.
- `http_status_codes`: Applicable HTTP status codes for this error.
- `suggested_application_actions`: Suggest practical actions that the developer of application consuming the API could take in order to resolve the error condition. These MUST be in English.
- `suggested_user_actions`: Suggest practical actions that a user of the application consuming the API could take in order to resolve the error condition. These MUST be in the language used `error_catalog.json#language`.
- `links`: Error context specific [HATEOAS](/docs/hypermedia.md) links. Corresponds to [`error.json#links`][22].
- `issues`: Issues associated with this error as defined in `error_spec_issue.json`. Each issue corresponds to an item in [`error_details.json`][28].

- [`error_spec_issue.json`][25] defines details related to the error. For example, there could be multiple validation errors triggering `400 BAD REQUEST`. Each invalid field MUST be listed in the [`error_details.json`][28] while sending the error response.
  - `id`: Catalog-unique identifier of the issue. This is required in order to search for the `error_spec_issue` from cached error catalog.
  - `issue`: Reason for error. This value MUST be the value set in [`error_details.json#issue`][28]. The issue string could have variables. Please use parametrized string following the Java String Formatter syntax. This value chould be localized.

<h3 id="string-format">String Format</h3>

For API services that are implemented in Java, various strings found in the error catalog MUST be formatted using Java's `printf-style` inspired format. It's recommended to use Java's [format specification](https://docs.oracle.com/javase/7/docs/api/java/util/Formatter.html#summary) for values of `message` and `issue` fields in the error catalog where applicable.

Service developers are strongly encouraged to use tools, such as [Java String Formatter](https://docs.oracle.com/javase/7/docs/api/java/util/Formatter.html) or similar, to interpret the formatted strings as found in the error catalog.

For example, an `error_spec` having value `Could not add card due to failure to comply with guideline %s` for `message` must be interpreted using a formatter as shown below.

```
com.foo.platform.error.ErrorSpec errorSpec = <find errorSpec from catalog>

com.foo.platform.error.Error error = new Error();
error.setName(errorSpec.getName());
error.setDebugId("debugId-777");
error.setLogLevel(errorSpec.getLogLevel());
String errorMessageString = String.format(errorSpec.getMessage(), "GUIDELINE: XYZ");
error.setMessage(errorMessageString);

List<Detail> details = new ArrayList<>();
Detail detail = new Detail();
String issueString = String.format(errorSpec.getIssues().get(0).getIssue(), ... <variables in issue string>);
detail.setIssue(issueString);
details.add(detail);
error.setDetails(details);

Response response = Response.status(Response.Status.BAD_REQUEST).entity(error).encoding(MediaType.APPLICATION_JSON).build();
throw new WebApplicationException(response);
```

The above example code is for illustration purposes only.

### Samples

This section provides some sample error catalogs.

#### Sample catalog : namespace : payments

```
{
 "namespace": "payments",
 "language": "en-US",
 "errors": [{
  "error_spec": {
   "name": "VALIDATION_ERROR",
   "message": "Invalid request - see details",
   "log_level": "ERROR",
   "http_status_codes": [
    400
   ],
   "issues": [{
    "id": "InvalidCreditCardType",
    "issue": "Value is invalid (must be visa, mastercard, amex, or discover)"
   }],
   "suggested_application_actions": [
    "Provide an acceptable card type and resend the request."
   ]
  }
 }, {
  "error_spec": {
   "name": "PAYEE_ACCOUNT_LOCKED_OR_CLOSED",
   "message": "Payee account is locked or closed",
   "log_level": "ERROR",
   "http_status_codes": [
    422
   ],
   "legacy_code": "PAYER_ACCOUNT_LOCKED_OR_CLOSED",
   "issues": [{
    "id": "PayerAccountLocked",
    "issue": "The account receiving this payment is locked or closed and cannot receive payments."
   }],
   "suggested_user_actions": [
    "Contact Customer Service at contact@foo.com"
   ]
  }
 }]
}
```

#### Sample catalog : namespace : wallet

```
{
 "namespace": "wallet",
 "language": "en-US",
 "errors": [{
  "error_spec": {
   "name": "INVALID_ISSUER_DETAILS",
   "message": "Invalid issuer details",
   "log_level": "ERROR",
   "http_status_codes": [
    400
   ],
   "issues": [{
    "id": "ISSUER_DATA_NOT_FOUND",
    "issue": "Issuer data not found"
   }],
   "suggested_application_actions": [
    "Provide issuer related data and resend the request."
   ]
  }
 }, {
  "error_spec": {
   "name": "INSTRUMENT_BLOCKED",
   "message": "Instrument is currently blocked.",
   "log_level": "ERROR",
   "http_status_codes": [
    422
   ],
   "issues": [{
    "id": "BankAccountBlocked",
    "issue": "Bank account is blocked due max random deposit retries. "
   }],
   "suggested_user_actions": [
    "Contact Customer Service at contact@foo.com."
   ]
  }
 }]
}
```

#### Sample catalog : namespace : payment-networks

```
{
 "namespace": "payment-networks",
 "language": "en-US",
 "errors": [{
  "error_spec": {
   "name": "VENDOR_TIMEOUT",
   "message": "Transaction timed out while waiting for response from downstream service provided by a 3rd party vendor.",
   "log_level": "ERROR",
   "http_status_codes": [
    504
   ],
   "suggested_application_actions": [
    "Retry again later."
   ]
  }
 }, {
  "error_spec": {
   "name": "INTERNAL_TIMEOUT",
   "message": "Internal error due to timeout. Request took too long to process. The status of the transaction is unknown.",
   "log_level": "ERROR",
   "http_status_codes": [
    500
   ],
   "suggested_application_actions": [
    "Contact Customer Service at contact@foo.com and provide Correlation-Id and debug_id for diagnosis along with other details."
   ]
  }
 }]
}
```