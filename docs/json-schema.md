# JSON SChema

We would assume that [JSON Schema](http://json-schema.org/) is used to describe request/response models. Determine the version of JSON Schema to use for your APIs. At the time of writing this, [draft-04][9] is the latest version. JSON Schema [draft-03][2] has been deprecated, as support in tools is mostly focused on draft-04. The draft-04 is backwards incompatible with draft-03.

## API Contract Description

There are various options available to define the API's contract interface (API specification or API description). Examples are: [OpenAPI (fka Swagger)][11], [Google Discovery Document](https://developers.google.com/discovery/v1/reference/apis#method_discovery_apis_getRest), [RAML](http://raml.org/), [API BluePrint](#https://apiblueprint.org/) and so on.

[OpenAPI][11] is a vendor neutral API description format. The OpenAPI [Schema Object][12] (or OpenAPI JSON) is based on the [draft-04][9] and uses a predefined subset of the draft-04 schema. In addition, there are extensions provided by the specification to allow for more complete documentation.

We have used OpenAPI wherever we need to describe the API specification throughout this document.

## $schema

**A note about using $schema with OpenAPI**

As of writing this (Q12017), OpenAPI tools DO NOT recognize `$schema` value and (incorrectly) assume the value of $schema to be `http://swagger.io/v2/schema.json#` only. The following description applies to JSON schema (<http://json-schema.org>) used in API specification specified using specifications other than OpenAPI.

Use [$schema](http://json-schema.org/latest/json-schema-core.html#anchor22) to indicate the version of JSON schema used by each JSON type you define as shown below.

```
{
    "type": "object",
    "$schema": "http://json-schema.org/draft-04/schema#",
    "name": "Order",
    "description": "An order transaction.",
    "properties": {

    }
}

```

In case your JSON type uses `links`, `media` and other such keywords or schemas such as for `linkDescription` that are defined in [http://json-schema.org/draft-04/hyper-schema](http://json-schema.org/draft-04/hyper-schema), you should provide the value for `$schema` accordingly as shown below.

```
{
    "type": "object",
    "$schema": "http://json-schema.org/draft-04/hyper-schema#",
    "name": "Linked-Order",
    "description": "An order transaction using HATEOAS links.",
    "properties": {

    }
}
```

If you are unsure about the specific schema version to refer to, it would be safe to refer `http://json-schema.org/draft-04/hyper-schema#` schema since it would cover all aspects of any JSON schema.

<h2 id="readOnly">readOnly</h2>

When resources contain immutable fields, `PUT`/`PATCH` operations can still be utilized to update that resource. To indicate immutable fields in the resource, the `readOnly` field can be specified on the immutable fields.

### Example

```
"properties": {
  "id": {
   "type": "string",
   "description": "Identifier of the resource.",
   "readOnly": true
  },
  }
```

## Migration From draft-03

Key differences between JSON schema [draft-03][10] and draft-04 have been captured by JSON Schema contributors in a [changelog](https://github.com/json-schema/json-schema/wiki/ChangeLog). Additionally, this [StackOverflow thread](http://stackoverflow.com/questions/17205260/json-schema-draft4-vs-json-schema-draft3) provides some additional quality migration guidance.

The following items are the most common migration issues API specifications will need to address:

#### required

draft-03 defines required fields in the composition of a field:

```
{
    "type": "object",
    "$schema": "http://json-schema.org/draft-03/schema#",
    "name": "Order",
    "description": "An order transaction.",
    "properties": {
        "id": {
            "type": "string",
            "description": "Identifier of the order transaction."
        },
        "amount": {
            "required": true,
            "description": "Amount being collected.",
            "$ref": "v1/schema/json/draft-04/amount.json"
        }
    }
}
```

In draft-04, an array as a peer to properties is used to designate the required fields:

```
{
    "type": "object",
    "$schema": "http://json-schema.org/draft-04/schema#",
    "name": "Order",
    "description": "An order transaction.",
    "required": [
        "amount"
    ],
    "properties": {
        "id": {
            "type": "string",
            "description": "Identifier of the order transaction."
        },
        "amount": {
            "description": "Amount being collected.",
            "$ref": "v1/schema/json/draft-04/amount.json"
        }
    }
}
```

#### readOnly

The `readonly` field was removed from draft-03, and replaced by [`readOnly`](http://json-schema.org/latest/json-schema-hypermedia.html#anchor15) (note the upper case `O`).

#### Format

The [format](http://tools.ietf.org/html/draft-zyp-json-schema-03#section-5.23) attribute is used when defining fields which fall into a certain predefined pattern.

These following formats are deprecated in draft-04:

- `date`
- `time`

Therefore, only `"format": "date-time"` MUST be used for any variation of date, time, or date and time. Any references to formats of `date` or `time` should be updated to `date-time`.

Field descriptions SHOULD indicate specifics of whether date or time is accepted. `date-time` specifies that ISO-8601/RFC3339 dates are utilized, which includes date-only and time-only.

## Advanced Syntax draft-04

Be aware that `anyOf`/`allOf`/`oneOf` syntax can cause issues with tooling, as code generators, documentation tools and validation of these keywords is often not implemented.

### allOf

The `allOf` keyword MUST only be used for the purposes listed here.

#### Extend object

The `allOf` keyword in JSON Schema SHOULD be used for extending objects. In draft-03, this was implemented with the `extends` keyword, which has been deprecated in draft-04.

##### Example

A common need is to extend a common type with additional fields. In this example, we will extend the [address](v1/schema/json/draft-04/address_portable.json) with a `type` field.

```
"shipping_address": { "$ref": "v1/schema/json/draft-04/address_portable.json" }
```

Using the `allOf` keyword, we can combine both the common type address schema and an extra schema snippet for the address type:

```
"shipping_address": {
  "allOf": [
    // Here, we include our "core" address schema...
    { "$ref": "v1/schema/json/draft-04/address_portable.json" },

    // ...and then extend it with stuff specific to a shipping
    // address
    { "properties": {
        "type": { "enum": [ "residential", "business" ] }
      },
      "required": ["type"]
    }
  ]
}
```

### anyOf/oneOf

The `anyOf` and `oneOf` keywords SHOULD NOT be used to design APIs. A variety of problems occur from these keywords:

- Codegen tools do not have accurate way to generate models/objects from these definitions.
- Developer portals would have significant difficulty in clearly explaining to API consumers the meaning of these relationships.
- Consumers using statically typed languages (e.g. C#, Java) have a more complex experience trying to conditionally deserialize objects which change based on some element.
  - Custom deserialization code is required to represent objects based on the response, standard libraries do not accommodate this out of the box.
  - Flat structures which combine all possible fields in an object are automatically deserialized properly.

##### anyOf/oneOf Problems: Example

```
{
    "activity_type": {
        "description": "The entity type of the item. One of 'PAYMENT', 'MONEY-REQUEST', 'RECURRING-PAYMENT-PROFILE', 'ORDER', 'PAYOUT', 'SUBSCRIPTION', 'INVOICE'",
        "type": "string",
        "enum": [
            "PAYMENT",
            "MONEY-REQUEST"
        ]
    },
    "extensions": {
        "type": "object",
        "description": "Extension to core activity fields",
        "oneOf": [
            {
                "$ref": "extended_properties.json#/definitions/payment_properties"
            },
            {
                "$ref": "extended_properties.json#/definitions/money_request_properties"
            }
        ]
    }
}
```

In order for an API consumer to deserialize this response (where POJO/POCO objects are used), standard mechanisms would not work. Because the `extensions` field can change on any given response, the consumer is forced to create a composite object to represent both `payment_properties.json` and `money_request_properties.json`.

A better approach:

```
{
    "activity_type": {
        "description": "The entity type of the item. One of 'PAYMENT', 'MONEY-REQUEST', 'RECURRING-PAYMENT-PROFILE', 'ORDER', 'PAYOUT', 'SUBSCRIPTION', 'INVOICE'",
        "type": "string",
        "enum": [
            "PAYMENT",
            "MONEY-REQUEST"
        ]
    },
    "payment": {
        "type": "object",
        "description": "Payment-specific activity.",
        "$ref": "payment.json"
    },
    "money_request": {
        "type": "object",
        "description": "Money request-specific activity.",
        "$ref": "money_request.json"
    }
}
```

In this scenario, both `payment` and `money_request` are in the definition. However, in practice, only one field would be serialized based on the `activity_type`.

For API consumers, this is a very predictable response, and allows for easy deserialization through standard libraries, without writing custom deserializers.
