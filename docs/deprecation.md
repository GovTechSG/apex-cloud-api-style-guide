# Deprecation

This document describes a solution to deprecate parts of an API as the API evolves. It is an extension to the API Versioning Policy.

### Terms Used

The term _`API Element`_ is used throughout this document to refer to the _things_ that could be deprecated in an API. Examples of an `API Element` are: an endpoint, a query parameter, a path parameter, a property within a JSON Object schema, JSON Object schema of a type or a custom HTTP header among other things.

The term _`old API`_ is used to indicate existing minor or major version of your API or an existing different API that your API supersedes.

The term _`new API`_ is used to indicate a new minor or major version of your API or a new different API that supersedes the `old API`.

_`API definition`_ is in the form of specification of an interface of a service following the [OpenAPI][11]. API's definition could be found in `swagger.json`.

## Background

When defining your API, you must make a lot of material decisions that have long lasting implications. The objective is to make a long-lived, durable, and reusable API. You are trying to get it "right". Practically speaking, however, you are not going to succeed every time. New requirements come in. Your understanding of the problem changes. What probably looked like a good decision at the time, may now limit your ability to elegantly extend your API to satisfy your new understanding. Lightweight decisions made in the past now seem somewhat heavy as the implications come into focus. Maintaining backward compatibility is a persistent challenge.

One option is to create a new major version of your API. This allows you to leave past decisions behind and start fresh. Unfortunately, it also means that all of your clients now need to migrate to new endpoints for any of the new work to deliver customer value. This is hard. Many clients will not move without good incentives. There is a lot of overhead in managing the customer migration. You also need to support two sets of interfaces for quite some time. The other consideration is that your API product may have multiple endpoints, but the breaking changes that you want to make only affect one. Making your customers migrate their applications for all the API endpoints just so you can “fix” one small part is a pretty heavyweight and expensive change. While pure and simple from a philosophical and engineering point of view, it is often unjustifiable from an ROI standpoint. The goal of this guideline is to find a middle ground that provides a more practical path forward when minor changes are needed, but which is still consistent, in spirit, with the [API Versioning Policy](#api-versioning-policy).

## Requirements

Here are the requirements for deprecation.

1. An API developer should be able to deprecate an `API Element` in a minor version of an API.
2. An API specification MUST highlight one or more deprecated elements of the API so the API consumers are aware.
3. An API server MUST inform client app(s) regarding deprecated elements present in request and/or response at runtime so that tools can recognize this, log warnings and highlight the usage of deprecated elements as needed.
4. Deprecated `API Elements` must remain supported for the life of the major version or until customers are no longer using them (the means to determine this are left to the discretion of the API owner since it's their customers who will ultimately be impacted).

## Solution

The following describes how to address the requirements listed above. The solution involves addressing documentation related requirement using an annotation and using a custom header to address the runtime related requirement.

1. [Documentation](#deprecation-documentation)
2. [Runtime](#deprecation-runtime)

### Documentation

An optional annotation named `x-deprecated` is used to mark an `API Element` as deprecated in the definition of the API.

#### Annotation: x-deprecated

`x-deprecated` can be used to deprecate any kind of `API Element`. The annotation should be used inline precisely where the `API Element` is defined. It is expected that the API tools generating documentation by introspecting the API definition would recognize this annotation and highlight the corresponding `API Element` as deprecated. It is also assumed that this annotation can be completely ignored by tools including those generating implementation bindings (POJO). In other words, it is not in scope of this solution that any implementation language specific construct (such as Java annotation `@deprecated`) would be generated for the `x-deprecated` annotation.

It is expected that the API documentation would highlight deprecated `API Elements` annotated by `x-deprecated` in the API specification distinctly and at the appropriate granularity.

### Schema for x-deprecated Annotation

We have provided specific JSON object types to use for deprecation of specific `API Elements`. This section lists schema for these types. The intent in providing schema for specific application of `x-annotation` is to make it easy for API developers to annotate the `API definition` and for API tool(s) to highlight each deprecated `API Element` with appropriate details.

##### Common Schema Elements

Following are common schema types that are used across new JSON object types to be used for deprecation.

```

  "x-deprecatedValue": {
   "type": "string",
   "description": "Value of the element that is deprecated. Use to deprecate a particular value in parameter or schema property as applicable."
  },
  "x-deprecatedSee": {
   "type": "string",
   "description": "URI (indirect or absolute) or name of to new parameter, resource, method, api_element, as applicable."
  },
  "x-apiVersion": {
   "pattern": "^[1-9][0-9]*[.][0-9]+$",
   "minLength": 3,
   "maxLength": 8,
   "description": "This string should contain the release or version number at which this schema element became deprecated. Version should be in the format '{major}.{minor}' (no leading 'v')."
  }

```

<h5 id="deprecation-schema-resource">Deprecated Resource</h5>

The following schema MUST be used to deprecate resource objects in `API definition`. Examples of resource object in `swagger.json` are: `operation` and `paths`.

```
  "x-deprecatedResource": {
   "type": "object",
   "title": "Schema for a deprecated resource.",
   "description": "Schema for deprecating a resource API element. A resource API element could be an operation or paths.",
   "properties": {
    "see": {
     "$ref": "#/definitions/x-deprecatedSee"
    },
    "since_version": {
     "$ref": "#/definitions/x-apiVersion"
    }
   }
  }
```

The following section provides several examples showing usage of `deprecatedResource` for `x-deprecated` annotation at resource level.

<h6 id="deprecation-example-resource">Example: Deprecated Resource</h6>

The following example shows deprecated resource named `commercial-entities` in `swagger.json`.

```
    "paths": {

        "/commercial-entities": {
             "x-deprecated": {
              "see": "financial-entities",
              "since_version": "1.4"
             },
        ...
        }

```

<h6 id="deprecation-example-method">Example: Deprecated Method</h6>

The following example shows a deprecated method `PUT /commercial-entities/{merchant_id}/agreements` and encourages to use new method `PATCH /commercial-entities/{merchant_id}/agreements`.

```
    "paths": {
        "/commercial-entities/{merchant_id}/agreements": {
            "put": {
                "summary": "Updates the Commercial Entity Agreements Details for a Merchant.",
                "operationId": "commercial-entity.agreement.update",
                "x-deprecated": {
               "see": "patch",
               "since_version": "1.4"
             },
             "parameters": [
                    {
                        "name": "merchant_id",
                        "in": "path",
                        "description": "The encrypted Merchant's identifier.",
                        "required": true,
                        "type": "string"
                    },
                    {
                        "name": "Agreements",
                        "in": "body",
                        "description": "An array of AgreementDetails",
                        "required": true,
                        "schema": {
                            "$ref": "./model/agreement_details.json"
                        }
                    }
                ],
            ...
         }
```

<h5 id="deprecation-schema-parameter">Deprecated Parameter</h5>

`swagger.json` provides a way to define one or more parameter for a method. The type of parameters are: path, query and header. Typically, query and header parameters can be deprecated. The following schema MUST be used while using `x-deprecated` annotation for a parameter.

```
  "x-deprecatedParameter": {
   "type": "object",
   "title": "Schema for a deprecated parameter.",
   "description": "Schema for deprecating an API element inline. The API element could be a custom HTTP header or a query param.",
   "properties": {
    "value": {
     "$ref": "#/definitions/x-deprecatedValue"
    },
    "see": {
     "$ref": "#/definitions/x-deprecatedSee"
    },
    "since_version": {
     "$ref": "#/definitions/x-apiVersion"
    }
   }
  }
```

The following section provides several examples showing usage of `deprecatedParameter` for `x-deprecated` annotation at parameter level.

<h6 id="deprecation-example-query-parameter">Example: Deprecated Query Parameter</h6>

The following example shows usage of the `x-deprecated` annotation in `swagger.json` to indicate deprecation of a query parameter `record_date`.

```
        "/commercial-entities/{merchant_id}": {
            "get": {
                "summary": "Gets a Commercial Entity as denoted by the merchant_id.",
                "description": "Gets the Commercial Entity as denoted by the merchant_id.",
                "operationId": "commercial-entity.get",
                "parameters": [
                    {
                        "name": "record_date",
                        "in": "query",
                        "description": "The date to use for the query; defaulted to yesterday.",
                        "required": false,
                        "type": "string",
                        "format": "date",
                        "x-deprecated": {
                            "since_version": "1.5",
                            "see": "transaction_date"
                        }
                    },
                    {
                        "name": "transaction_date",
                        "in": "query",
                        "description": "The date to use for the query; defaulted to yesterday.",
                        "required": false,
                        "type": "string",
                        "format": "date"
                    },
                    ...
```

<h6 id="deprecation-example-header">Example: Deprecated Header</h6>

The following example shows a deprecated custom HTTP header called `CLIENT_INFO`.

###### OpenAPI

```
        "/commercial-entities/{merchant_id}": {
            "get": {
                "summary": "Gets a Commercial Entity as denoted by the merchant_id.",
                "description": "Gets the Commercial Entity as denoted by the merchant_id.",
                "operationId": "commercial-entity.get",
                "parameters": [
                    ...
                    {
                        "name": "CLIENT_INFO",
                        "in": "header",
                        "description": "Optional header for all the API's to pass on api caller tracking information. This header helps capture any input from the caller service and pass it along to analytics for tracking .",
                        "in" : "header",
                        "x-deprecated": {
                             "since_version": "1.5"
                        }

                    }
                    ...
```

<h6 id="deprecation-example-query-parameter-value">Example: Deprecated Query Parameter Value</h6>

The following example shows usage of the `x-deprecated` annotation in `swagger.json` to indicate deprecation of a specific value (`y`) for a query parameter named `fields`.

```
        "/commercial-entities": {
            "get": {
                "summary": "Gets Commercial Entities.",
                "description": "Gets the Commercial Entities.",
                "operationId": "commercial-entity.get",
                "parameters": [
                    {
                        "name": "fields",
                        "in": "query",
                        "description": "Fields to return in response, default is x, possible values are x, y, z.",
                        "required": false,
                        "type": "string",
                        "x-deprecated": {
                            "since_version": "1.5",
                            "value": "y"
                        }
                    },
                    {
                        "name": "transaction_date",
                        "in": "query",
                        "description": "The date to use for the query; defaulted to yesterday.",
                        "required": false,
                        "type": "string",
                        "format": "date"
                    },
                    ...
```

<h5 id="deprecation-schema-object">Deprecated JSON Object Schema</h5>

In order to deprecate a schema of JSON Object itself or one or more properties within the JSON Object schema, we recommend using a schema called `deprecatedSchema` for the `x-deprecated` annotation.

```
  "x-deprecatedSchema": {
   "type": "array",
   "description": "Schema for a collection of deprecated items in a schema.",
   "items": {
    "$ref": "#/definitions/x-deprecatedSchemaProperty"
   }
  }
```

```
  "x-deprecatedSchemaProperty": {
   "type": "object",
   "title": "Schema for a deprecated schema property or schema itself.",
   "description": "Schema for deprecating an API element within JSON Object schema. The API element could be an individual property of a schema of a JSON type or an entire schema representing JSON object.",
   "required": ["api_element"],
   "properties": {
    "api_element": {
     "type": "string",
     "description": "JSON pointer to API element that is deprecated. If the API element is JSON Object schema of a type itself, JSON pointer MUST point to the root of that schema. If the API element is a property of schema, the JSON pointer MUST point to that property."
    },
    "value": {
     "$ref": "#/definitions/x-deprecatedValue"
    },
    "see": {
     "$ref": "#/definitions/x-deprecatedSee"
    },
    "since_version": {
     "$ref": "#/definitions/x-apiVersion"
    }
   }
  }

```

In order to avoid extending JSON draft-04 schema with `keywords` needed to either deprecate the schema itself by using `x-annotation` in metadata section of the schema or deprecate individual properties inline, we have chosen a less disruptive route of using `x-annotation` next to references for JSON Object in `API definition`. Therefore, this annotation SHOULD be used in `API definition` where schema is "referred". In case if you are OK with inlining, you should go ahead and annotate individual deprecated properties in schema or schema itself.

As of March 2017, [OpenAPI 3.0.0-rc0](https://github.com/OAI/OpenAPI-Specification/blob/OpenAPI.next/versions/3.0.md) has introduced `deprecated` flag to apply at operation, parameter and schema field levels. `x-annotation` could be used alongside the `deprecated` flag to provide additional useful information for the deprecated `API Element`.

<h6 id="deprecation-example-schema-property">Example: Deprecated Property In Response</h6>

The following example shows usage of the `x-deprecated` annotation in `API definitions` to indicate deprecation of a property named `address` in response.

```
        "/commercial-entities/{merchant_id}": {
            "get": {
                "summary": "Gets a Commercial Entity as denoted by the merchant_id.",
                "description": "Gets the Commercial Entity as denoted by the merchant_id.",
                "operationId": "commercial-entity.get",
                "responses": {
                    "200": {
                        "description": "The Commercial Entity.",
                        "schema": {
                            "$ref": "./model/interaction/commercial-entities/merchant_id/get_response.json",
                            "x-deprecated": [
                    {
                    "api_element": "./model/interaction/commercial-entities/merchant_id/get_response.json#/address",
                    "see": "./model/interaction/commercial-entities/merchant_id/get_response.json#/global_address",
                    "since_version": "1.4"
                    }
                    ]
                        }
                    },
                    "default": {
                        "description": "Unexpected error",
                        "schema": {
                            "$ref": "v1/schema/json/draft-04/error.json"
                        }
                    }
                }
```

<h6 id="deprecation-example-enum">Example: Deprecated Enum In Response</h6>

The following example shows usage of the `x-deprecated` annotation in `API definitions` to indicate deprecation of an enum `FAILED` used by a property named `state`.

```
       "/commercial-entities/{merchant_id}": {
            "get": {
                "summary": "Gets a Commercial Entity as denoted by the specified merchant identifier.",
                "description": "Gets the Commercial Entity as denoted by the specified merchant identifier.",
                "operationId": "commercial-entity.get",
                "responses": {
                    "200": {
                        "description": "The Commercial Entity.",
                        "schema": {
                            "$ref": "./model/interaction/commercial-entities/merchant_id/get_response.json",
                            "x-deprecated": [
                    {
                    "api_element": "./model/interaction/commercial-entities/merchant_id/get_response.json#/state",
                    "value": "FAILED",
                    "since_version": "1.4"
                    }
                    ]
                        }
                    },
                    "default": {
                        "description": "Unexpected error",
                        "schema": {
                            "$ref": "v1/schema/json/draft-04/error.json"
                        }
                    }
                }


```

### Runtime

The API server MUST inform client app(s) of the deprecated `API element`(s) present in request and/or response.

<h4 id="deprecation-header">Header: Foo-Deprecated</h4>

It is recommended to use a custom HTTP header named `Foo-Deprecated` to convey deprecation related information. The service MUST respond with the `Foo-Deprecated` header in the following cases:

1. The caller has used one or more deprecated element in request.
2. There is one or more deprecated element in the response.

In order to avoid bloating the responses with static information related to deprecation that does not change from response to response on the same end point, we recommend that API developers provide just an empty JSON object as a value as shown below. In the future, we plan to replace this with something that still does not bloat the responses but still provides enough information so that tools could scan responses for deprecation and take appropriate actions such as notifying App developers/administrators.

```
"Foo-Deprecated": "{}"
```

**Note**: Applications consuming this header MUST not take any action based on the value of the header at this time. Instead, we recommend that these applications SHOULD take action based only on the existence of the header in the response.