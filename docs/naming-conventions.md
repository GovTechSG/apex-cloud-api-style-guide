
# Naming Conventions

Naming conventions for URIs, query parameters, resources, fields and enums are described in this section. Let us emphasize here that these guidelines are less about following the conventions exactly as described here but they are more about defining some naming conventions and sticking to them in a consistent manner while designing APIs. For example, we have followed [snake_case](https://en.wikipedia.org/wiki/Snake_case) for field and file names, however, you could use other forms such as [CamelCase](https://en.wikipedia.org/wiki/Camel_case) or something else that you have devised yourself. It is important to adhere to a defined convention.

## URI Component Names

URIs follow [RFC 3986][8] specification. This specification simplifies REST API service development and consumption. The guidelines in this section govern your URI structure and semantics following the RFC 3986 constraints.

### URI Components

According to RFC 3986, the generic URI syntax consists of a hierarchical sequence of components referred to as the scheme, authority, path, query, and fragment as shown in example below.

```
         https://example.com:8042/over/there?name=ferret#nose
         \___/   \_______________/\_________/\_________/\__/
          |           |            |            |        |
       scheme     authority       path        query   fragment
```

### URI Naming Conventions

Following is a brief description of the URI specific naming convention guidelines for APIs. This specification uses parentheses "( )" to group, an asterisk " \* " to specify zero or more occurrences, and brackets "[ ]" for optional fields.

```
[scheme"://"][host[':'port]]"/v" major-version '/'namespace '/'resource ('/'resource)* '?' query

```

- URIs MUST start with a letter and use only lower-case letters.
- Literals/expressions in URI paths SHOULD be separated using a hyphen ( - ).
- Literals/expressions in query strings SHOULD be separated using underscore ( \_ ).
- URI paths and query strings MUST percent encode data into [UTF-8 octets](https://en.wikipedia.org/wiki/Percent-encoding).
- Plural nouns SHOULD be used in the URI where appropriate to identify collections of data resources.
  - `/invoices`
  - `/statements`
- An individual resource in a collection of resources MAY exist directly beneath the collection URI.
  - `/invoices/{invoice_id}`
- Sub-resource collections MAY exist directly beneath an individual resource. This should convey a relationship to another collection of resources (invoice-items, in this example).
  - `/invoices/{invoice_id}/items`
- Sub-resource individual resources MAY exist, but should be avoided in favor of top-level resources.
  - `/invoices/{invoice_id}/items/{item_id}`
  - Better: `/invoice-items/{invoice_item_id}`
- Resource identifiers SHOULD follow [recommendations](/docs/uri?id=resource-identifiers) described in subsequent section.

**Examples**

- `https://api.foo.com/v1/vault/credit-cards`
- `https://api.foo.com/v1/vault/credit-cards/CARD-7LT50814996943336KESEVWA`
- `https://api.foo.com/v1/payments/billing-agreements/I-V8SSE9WLJGY6/re-activate`

**Formal Definition:**

| Term          | Defiition                                                                   |
| ------------- | --------------------------------------------------------------------------- |
| URI           | [end-point] '**/**' resource-path ['**?**'query]                            |
| end-point     | [scheme "**://**"] host ['**:**' port]]                                     |
| scheme        | "**http**" or "**https**"                                                   |
| resource-path | "**/v**" version '**/**' namespace-name '**/**' resource ('**/**' resource) |
| resource      | resource-name ['**/**' resource-id]                                         |
| resource-name | Alpha (Alpha \| Digit \| '-')\*                                             |
| resource-id   | value                                                                       |
| query         | name '**=**' value ('**&**' name = value)\*                                 |
| name          | Alpha (Alpha \| Digit \| '\_')\*                                            |
| value         | URI Percent encoded value                                                   |

Legend

```
'   Surround a special character with single quotes
" Surround strings with double quotes
() Use parentheses for grouping
[] Use brackets to specify optional expressions
* An expression can be repeated zero or more times
```

### Resource Names

When modeling a service as a set of resources, developers MUST follow these principles:

- Nouns MUST be used, not verbs.
- Resource names MUST be singular for singletons; collections' names MUST be plural.

  - A description of the automatic payments configuration on a user's account
    - `GET /autopay` returns the full representation
  - A collection of hypothetical charges:
    - `GET /charges` returns a list of charges that have been made
    - `POST /charges` creates a new charge resource, `/charges/1234`
    - `GET /charges/1234` returns a full representation of a single charge

- Resource names MUST be lower-case and use only alphanumeric characters and hyphens.
  - The hyphen character, ( - ), MUST be used as a word separator in URI path literals. **Note** that this is the only place where hyphens are used as a word separator. In nearly all other situations, the underscore character, ( \_ ), MUST be used.

### Query Parameter Names

- Literals/expressions in query strings SHOULD be separated using underscore ( \_ ).
- Query parameters values MUST be percent-encoded.
- Query parameters MUST start with a letter and SHOULD be all in lower case. Only alpha characters, digits and the underscore ( \_ ) character SHALL be used.
- Query parameters SHOULD be optional.
- Some query parameter names are reserved, as indicated in [Resource Collections](#resource-collections).

For more specific info on the query parameter usage, see [URI Standards](#uri-standards-query).

## Field Names

The data model for the representation MUST conform to [JSON][30]. The values may themselves be objects, strings, numbers, booleans, or arrays of objects.

- Key names MUST be lower-case words, separated by an underscore character, ( \_ ).
  - `foo`
  - `bar_baz`
- Prefix such as `is_` or `has_` SHOULD NOT be used for keys of type boolean.
- Fields that represent arrays SHOULD be named using plural nouns (e.g. authenticators-contains one or more authenticators, products-contains one or more products).

## Enum Names

Entries (values) of an enum SHOULD be composed of only upper-case alphanumeric characters and the underscore character, ( \_ ).

- `FIELD_10`
- `NOT_EQUAL`

If there is an industry standard that requires us to do otherwise, enums MAY contain other characters.

## Link Relation Names

A link relation type represented by `rel` must be in lower-case.

- Example

```
"links": [
    {
        "href": "https://uri.foo.com/v1/customer/partner-referrals/ALT-JFWXHGUV7VI/activate",
        "rel": "activate",
        "method": "POST"

    }
   ]
```

## File Names

JSON schema for various types used by API SHOULD each be contained in separate files, referenced using the `$ref` syntax (e.g. `"$ref":"object.json"`). JSON Schema files SHOULD use underscore naming syntax, e.g. `transaction_history.json`.
