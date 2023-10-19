# JSON Types

This section provides guidelines related to usage of [JSON primitive types](#json-primitive) as well as [commonly useful JSON types](#common-types) for address, name, currency, money, country, phone, among other things.

## JSON Primitive Types

JSON Schema [draft-04][9] SHOULD be used to define all fields in APIs. As such, the following notes about the JSON Schema primitive types SHOULD be respected. Following are the guidelines governing use of JSON primitive type representations.

### String

At a minimum, `strings` SHOULD always explicitly define a `minLength` and `maxLength`.

There are several reasons for doing so.

1. Without a maximum length, it is impossible to reliably define a database column to store a given string.
2. Without a maximum and minimum, it is also impossible to predict whether a change in length will break backwards-compatibility with existing clients.
3. Finally, without a minimum length, it is often possible for clients to send an empty string when they should not be allowed to do so.

APIs MAY avoid defining `minLength` and `maxLength` only if the string value is from another upstream system that has refused to provide any information on these values. This decision must be documented in the schema.

API authors SHOULD consider practical limitations when defining `maxLength`. For example, when using the VARCHAR2 type, modern versions of Oracle can safely store a Unicode string of no more than 1,000 characters. (The size limit is [4,000](https://docs.oracle.com/cd/B28359_01/server.111/b28320/limits001.htm) bytes and each Unicode character may take up to four bytes for storage).

`string` SHOULD utilize the `pattern` property as appropriate, especially when defining enumerated values or numbers. However, it is RECOMMENDED not to overly constrain fields without a valid technical reason.

### Enumeration

The JSON Schema `enum` keyword is difficult to use safely. It is not possible to add new values to an `enum` in a schema that describes a service response without breaking backwards compatibility. In that scenario, clients will often reject responses with values that are not in the older copy of the schema that they posess. This is usually not the desired behavior. Clients should usually handle unknown values more gracefully, but since you can't control nor verify their behavior, it is not safe to add new enum values.

For the reasons stated above, the schema author MUST comply with the following guidelines while using an `enum` with the JSON type `string`.

- The keyword `enum` SHOULD be used only when the set of values are fixed and would never change in future.
- If you anticipate adding new values to the `enum` array in future, avoid using the keyword `enum`. You SHOULD instead use a `string` type and document all acceptable values for the `string`. When using a `string` type to express enumerated values, you SHOULD enforce [naming conventions](/docs/naming-conventions.md) through a `pattern` field.
- If there is no technical reason to do otherwise -- for instance, a pre-existing database column of smaller size -- `maxLength` should be set to 255. `minLength` should be set to 1 to prevent clients sending the empty string.
- All possible values of an `enum` field SHOULD be precisely defined in the documentation. If there is not enough space in the `description` field to do so, you SHOULD use the API’s user guide to define them.

Given below is the JSON snippet for enforcing naming conventions and length constraints.

```
{
    "type": "string",
    "minLength": 1,
    "maxLength": 255,
    "pattern": "^[0-9A-Z_]+$",
    "description": "A description of the field. The possible values are OPTION_ONE and OPTION_TWO."
}

```

### Number

There are a number of difficulties associated with `number` type in JSON.

JSON itself defines a `number` very simply: it is an unbounded, fixed-point value. This is illustrated well by the railroad diagram for `number` at [JSON][30]. There is only one `number` type in JSON; there is no separate `integer` type.

JSON Schema diverges from JSON and defines two number types: `number` and `integer`. This is purely a convenience for schema validation; the JSON number type is used to implement both. Just as in JSON, both types are unbounded unless the schema author provides explicit `minimum` and `maximum` values.

Many programming languages do not safely handle unbounded values in JSON. JavaScript is an excellent example. A JSON deserializer is provided as part of the [ECMAScript](http://www.ecma-international.org/publications/standards/Ecma-262.htm) specification. However, it requires that all JSON numbers are deserialized into the only `number` type supported by JavaScript -- 64-bit floating point. This means that attempting to deserialize any JSON number larger than about 2^53 in a JavaScript client will result in an exception.

To ensure the greatest degree of cross-client compatibility possible, schema authors SHOULD:

- Never use the JSON Schema `number` type. Some languages may interpret it as a fixed-point value, and some as floating-point. Always use `string` to represent a decimal value.

- Only define `integer` types for values that can be represented in a 32-bit signed integer, that is to say, values between (`(2^31) - 1) and -(2^31`). This ensures compatibility across a wide range of programming languages and circumstances. For example, array indices in JavaScript are signed 32-bit integers.

- When using an `integer` type, always provide an explicit minimum and a maximum. This not only allows backwards-incompatible changes to be detected, it also guarantees that all values can fit in a 32-bit signed integer. If there are no technical reasons to do otherwise, the `maximum` and `minimum` should be defined as `2147483647 (((2^31) - 1))` and `-2147483648 (-(2^31))` or `0` respectively. Common sense should be used to determine whether to allow negative values. Business logic that could change in the future generally SHOULD NOT be used to determine boundaries; business rules can easily change and break backwards compatibility.

If there is any possibility that the value could not be represented by a signed 32-bit integer, now or in the future, not use the JSON Schema `integer` type. Use a `string` instead.

**Examples**

This `integer` type might be used to define an array index or page count, or perhaps the number of months an account has been open.

```
{
    "type": "integer",
    "minimum": 0,
    "maximum": 2147483647
}
```

When using a `string` type to represent a number, authors MUST provide boundaries on size using `minLength` and `maxLength`, and constrain the definition of the string to only represent numbers using `pattern`.

For example, this definition only allows positive integers and zero, with a maximum value of 999999:

```
{
    "type": "string",
    "pattern": "^[0-9]+$",
    "minLength": 1,
    "maxLength": 6
}
```

The following definition allows the representation of fixed-point decimal values both positive or negative, with a maximum length of 32 and no requirements on scale:

```
{
    "type": "string",
    "pattern": "^(-?[0-9]+|-?([0-9]+)?[.][0-9]+)$"
    "maxLength": 32,
    "minLength": 1,
}
```

### Array

JSON defines `array` as unbounded.

Although practical limits are often much lower due to memory constraints, many programming languages do place maximum theoretical limits on the size of arrays. For example, JavaScript is limited to the length of a 32-bit unsigned integer by the ECMA-262 specification. Java is [limited to about `Integer.MAX_VALUE - 8`](https://stackoverflow.com/questions/3038392/do-java-arrays-have-a-maximum-size#3039805), which is less than half of JavaScript.

To ensure maximum compatibility across languages and encourage paginated APIs, `maxItems` SHOULD always be defined by schema authors. `maxItems` SHOULD NOT have a value greater than can be represented by a 16-bit signed integer, in other words, `32767` or `(2^15) - 1)`.

Note that developers MAY choose to set a smaller value; the value `32767` is a default choice to be used when no better choice is available. However, developers SHOULD design their API for growth. For example, although a paginated API may only support a maximum of 100 results per page today, performance improvements may allow deveopers to improve that to 1,000 results next year. Therefore, `maxItems` SHOULD NOT be used to communicate maximum page size.

`minItems` SHOULD also be defined. In most situations, its value will be either `0` or `1`.

### Null

APIs MUST NOT produce or consume `null` values.

`null` is a primitive type in JSON. When validating a JSON document against JSON Schema, a property's value can be `null` only when it is explicitly allowed by the schema, using the `type` keyword (e.g. `{"type": "null"}`). Since in an API `type` will always need to be defined to some other value such as `object` or `string`, and these standards prohibit using schema composition keywords such as `anyOf` or `oneOf` that allow multiple types, if an API produces or consumes null values, it is unlikely that, according to the API's own schemas, this is actually valid data.

In addition, in JSON, a property that doesn't exist or is missing in the object is considered to be undefined; this is conceptually separate from a property that is defined with a value of `null`, but many programming languages have difficulty drawing this distinction.

For example, a property `my_property` defined as `{"type": "null"}` is represented as

```
{
    "my_property": null
}
```

While a property that is `undefined` would not be present in the object:

```
{ }
```

In most strongly typed languages, such as Java, there is no concept of an `undefined` type, which means that for all undefined fields in a JSON object, a deserializer would return the value of such types as `null` when you try to retrieve them. Similarly, some Java-based JSON serializers serialize fields to JSON `null` by default, even though it is not possible for the serializer to determine whether the author of the Java object intended for that property to be defined with a value of `null`, or simply undefined. In Jackson, for example, this behavior is moderated through use of the [`JsonInclude`](https://fasterxml.github.io/jackson-annotations/javadoc/2.8/com/fasterxml/jackson/annotation/JsonInclude.html) annotation. On the other hand, the org.json library defines an object called [NULL](https://stleary.github.io/JSON-java/org/json/JSONObject.html#NULL) to distinguish between `null` and `undefined`.

Eschewing JSON null completely helps avoid many of these subtle cross-language compatibility traps.

### Additional Properties

Setting of [`additionalProperties`](http://json-schema.org/latest/json-schema-validation.html#anchor134) to `false` in schema objects breaks backward compatibility in those clients that use an API's JSON schemas (defined by its contract) to validate the API requests and responses. For the same reason, the schema author MUST not explicitly set the `additionalProperties` to `false`.

The API implementation SHOULD instead enforce the conformance of requests and responses to an API contract by hard validating the requests and responses against the defined API contract at run-time.

## Common Types

Resource representations in API MUST reuse the [common data type][13] definitions where possible. Following sections provide some details about some of these common types. Please refer to the [`README`][14]and the [schema][13] for more details.

### Address

We recommend using [`address_portable.json`][19] for all requirements related to address. The `address_portable.json` is

- backward compatible with [hcard](http://microformats.org/wiki/hcard) address microformats,
- forward compatible with Google open-source address validation metadata ([i18n-api][10]) and W3 HTML5.1 [autofill][11] fields,
- allows mapping to and from many address normalization services (ANS) such as [AddressDoctor][16].

Please refer to [README for Address][12] for more details about the address type, guidance on how to map it to i18n-api's address and W3 HTML5.1's autofill fields.

### Money

Money is a standard type to represent amounts. The Common Type [`money.json`][15] provides common definition of money.

**Data-type integrity rules:**

- Both `currency_code` and `value` MUST exist for this type to be valid.
- Some currencies such as "JPY" do not have sub-currency, which means the decimal portion of the value should be ".0".
- An amount MUST NOT be negative. For example a $5 bill is never negative. Negative or positive is an internal notion in association with a particular account/transaction and in respect of the type of the transaction performed.

### Percentage, Interest rate, or APR

Percentages and interest rates are very common when dealing with money. One of the most common examples is annual percentage rate, or APR. These interest rates SHOULD always be represented according to the following rules:

- The Common Type [`percentage.json`](v1/schema/json/draft-04/percentage.json) MUST be used. This ensures that the rate is represented as a fixed-point decimal.
  - All validation rules defined in the type MUST be followed.
- The value MUST be represented as a percentage.
  - **Example:** if the interest rate is 19.99%, the value returned by the API MUST be `19.99`.
- The field's JSON schema description field SHOULD inform clients how the representation works.
  - **Example:** "The interest rate is represented as a percentage. For example, an interest rate of 19.99% would be serialized as 19.99."
- It is the responsibility of the client to transform this value into a format suitable for display to the end-user. For example, some countries use the comma ( , ) as a decimal separator instead of the period ( . ). Services MUST NOT vary the format of values passed to or from a service based on end-user display concerns.

### Internationalization

The following common types MUST be used with regard to global country, currency, language and locale.

- [`country_code`](v1/schema/json/draft-04/country_code.json)
  - All APIs and services MUST use the [ISO 3166-1 alpha-2](http://www.iso.org/iso/country_codes.htm) two letter country code standard.
- [`currency_code`](v1/schema/json/draft-04/currency_code.json)
  - Currency type MUST use the three letter currency code as defined in [ISO 4217](http://www.currency-iso.org/). For quick reference on currency codes, see [http://en.wikipedia.org/wiki/ISO_4217](http://en.wikipedia.org/wiki/ISO_4217).
- [`language.json`](v1/schema/json/draft-04/language.json)
  - Language type uses [BCP-47](https://tools.ietf.org/html/bcp47) language tag.
- [`locale.json`](v1/schema/json/draft-04/locale.json)
  - Locale type defines the concept of locale, which is composed of `country_code` and `language`. Optionally, IANA timezone can be included to further define the locale.
- [`province.json`](v1/schema/json/draft-04/province.json)
  - Province type provides detailed definition of province or state, based on [ISO-3166-2](https://en.wikipedia.org/wiki/ISO_3166-2) country subdivisions, with room for variant local, international, and abbreviated representations of province names. Useful for logistics, statistics, and building state pull-downs for on-boarding.

<h3 id="date-time">Date, Time and Timezone</h3>

When dealing with date and time, all APIs MUST conform to the following guidelines.

- The date and time string MUST conform to the `date-time` universal format defined in section `5.6` of [RFC3339][21]. In use cases where you would require only a subset of the fields (e.g `full-date` or `full-time`) from the RFC3339 `date-time` format, you SHOULD use the [Date Time Common Types](#date-common-types) to express these.

- All APIs MUST only emit [UTC](https://en.wikipedia.org/wiki/Coordinated_Universal_Time) time (aka [Zulu time](https://en.wikipedia.org/wiki/List_of_military_time_zones) or [GMT](https://en.wikipedia.org/wiki/Greenwich_Mean_Time)) in the responses.

- When processing requests, an API SHOULD accept `date-time` or time fields that contain an offset from UTC. For example, `2016-09-28T18:30:41.000+05:00` SHOULD be accepted as equivalent to `2016-09-28T13:30:41.000Z`. This helps ensure compatibility with third parties who may not be capable of normalizing values to UTC before sending requests. In such cases the offset SHOULD only be used to calculate the equivalent UTC time before it is persisted in the system (because of known platform/language/DB interoperability issues). A UTC offset MUST NOT be used to derive anything else.

- If the business logic requires expressing the timezone of an event, it is RECOMMENDED that you capture the timezone explicitly by using a separate request/response field. You SHOULD NOT use offset to derive the timezone information. The offset alone is insufficient to accurately transform the stored UTC time back to a local time later. The reason is that a UTC offset might be same for many geographical regions and based on the time of the year there may be additional factors such as daylight savings. For example, an offset UTC-05:00 represents Eastern Standard Time during winter, Central Dayight Time during summer, and year-round offset for Panama, Columbia, and Peru.

- The timezone string MUST be per [IANA timezone database](https://www.iana.org/time-zones) (aka **Olson** database or **tzdata** or **zoneinfo** database), for example _America/Los_Angeles_ for Pacific Time, or _Europe/Berlin_ for Central European Time.

- When expressing [floating](https://www.w3.org/International/wiki/FloatingTime) time values that are not tied to specific time zones such as user's date of birth, expiry date, publication date etc. in requests or responses, an API SHOULD NOT associate it with a timezone. The reason is that a UTC offset changes the meaning of a floating time value. For examples, all countries with timezones west of prime meridian would consider a floating time value to be the previous day.

#### Date Time Common Types

The following common types MUST be used to express various date-time formats:

- [`date_time.json`](v1/schema/json/draft-04/date_time.json) SHOULD be used to express an RFC3339 `date-time`.
- [`date_no_time.json`](v1/schema/json/draft-04/date_no_time.json) SHOULD be used to express `full-date` from RFC 3339.
- [`time_nodate.json`](v1/schema/json/draft-04/time_nodate.json) SHOULD be used to express `full-time` from RFC3339.
- [`date_year_month.json`](v1/schema/json/draft-04/date_year_month.json) SHOULD be used to express a floating date that contains only the **month** and **year**. For example, card expiry date (`2016-09`).
- [`time_zone.json`](v1/schema/json/draft-04/time_zone.json) SHOULD be used for expressing timezone of a RFC3339 `date-time` or a `full-time` field.