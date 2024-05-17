---
title: Kiota API models
description: Learn how OpenAPI types are used to generate models for Kiota API clients.
author: baywet
ms.author: vibiret
ms.topic: conceptual
date: 03/21/2023
---

# Kiota API models

The primary goal of models is to enable a developer to easily craft request payloads and get result as high levels objects without having to define their own types and/or implement serialization/deserialization themselves.

## Components versus inline models

All models declared as components will be generated under a `models` sub-namespace. If models/components are namespaced `components/modelA`, `components/ns1/modelB`, the namespaces will be included as sub-namespaces of the `models` namespace.

All models declared inline with an operation will be generated under the namespace of the operation, next to the request builder it is used by. As a reminder, all request builders are put in namespaces according to the path segment the refer to.

## Inheritance

Models in an [allOf](https://spec.openapis.org/oas/latest.html#composition-and-inheritance-polymorphism) schema declaration will inherit from each other. Where the uppermost type in the collection is the greatest ancestor of the chain.

### allOf interpretation rules

The following table describes how kiota will project any schema with allOf entries when processing an API description:

| Number of Properties | Number of allOf entries without a reference / inline | Number of allOf entries with a reference | Total number of allOf entries | Result |
| -------------------- | ---------------------------------------- | ------------------------------------------- | ----------------------------- | ------ |
| 0 | 0 | 0 | 0 | Ignored/Invalid |
| 0 | 0 or 1 | 0 or 1 | 1 | Ignored, will process the only allOf entry instead and use the original schema's description and name. |
| 1 or more | 0 | 0 | 0 | Class/interface without a parent type. |
| 0 | 1 | 1 | 2 | Class/interface with properties from the inline schema and a parent type from the referenced schema. |
| 1 or more | 0 | 1 | 1 | Class/interface with properties from the current schema and a parent type from the referenced schema. |
| 1 or more | 1 | 0 | 1 | Class/interface with properties from the current schema and a parent type from the inline schema. |
| 0 | 0 | 1 | 1 | Class/interface with properties from the referenced schema without a parent type. |
| 0 | 1 | 0 | 1 | Class/interface with properties from the inline schema without a parent type. |
| 1 or more | 1 | 1 | 2 | Class/interface with properties from the current schema and a parent type from referenced schema if it has properties. |
| 1 or more | 1 | 1 | 2 | Class/interface with properties from the current schema and a parent type from the inline schema if it has properties. |
| 0 | 0 | 0 | 0 | Invalid scenario for inheritance |
| 1 or more | 1 or more | 1 or more | * | Invalid scenario for inheritance. Will result in a class/interface will all the properties from the current schema as well as all the properties from the allOf entries. |

> [!NOTE]
> These rules are applied to allOf entries recursively enabling multi-level inheritance.

## Faceted implementation of oneOf

`oneOf` specifies a type union (exclusive) where the response can be of one of the specified child schemas. Kiota implements that specification by generating types for all the child schemas and using a union type for languages that support it or a wrapper type with one property per type in the union.

The deserialized result will either be of the one of the types of the union or be of the wrapper type with only one of the properties being non-null.

When a `oneOf` keyword has at least one child schema that is of type object then the OpenAPI discriminator keyword MUST be provided to identify the applicable schema.

Child schemas that are arrays or primitives will use the equivalent type language parser to attempt to interpret the input value. The first primitive schema that does not fail to parse will be used to deserialize the input.

Nested `oneOf` keywords are only supported when the child schema uses a `$ref` to enable naming the nested type.

## Faceted implementation of anyOf

`anyOf` specifies a type intersection (inclusive union) where the response can be of any of the specified child schemas. Kiota implements that specification by generating types for all the child schemas and using a intersection type for languages that support it or a wrapper type with one property per type in the union.

The deserialized result will either intersection type or be of the wrapper type with one or more of the properties being non-null.

Where there are common properties in the child schemas, the corresponding value in the input will deserialized into first the child schemas with the common property.

## Heterogeneous collections

For any collection of items that rely on `allOf`, `anyOf`, or `oneOf`, it is possible the result will contain multiple types of objects.

For example, think of an endpoint returning a collection of directory objects (abstract). Directory object is derived by User and Group, which each have their own set of properties. In this case the endpoint will be documented as returning a collection of directory objects and return in reality a mix of users and groups.

Kiota supports discriminators by down-casting the returned object during deserialization. The down-casting is supported through the use of `allOf` and discriminator property. Kiota supports both [implicit and explicit discriminator mappings](#example-1---using-allof-and-discriminator). [Using `oneOf` to constrain derived types](#example-2---using-oneof-to-constrain-the-derived-types---not-supported) is **not** supported as it will be interpreted as an intersection type.

In case of inline schemas, the type will be named by conventions:

- Last segment name + operation + requestBody
- Last segment name + operation + response
- Parent type name + member + id (sequential)

### Example 1 - using allOf and discriminator

```json
{
  "microsoft.graph.directoryObject": {
    "allOf": [
      { "$ref": "#/components/schemas/microsoft.graph.entity" },
      {
        "title": "directoryObject",
        "required": ["@odata.type"],
        "type": "object",
        "properties": {
          "@odata.type": {
            "type": "string",
            "default": "#microsoft.graph.directoryObject"
          }
        }
      }
    ],
    "discriminator": {
      "propertyName": "@odata.type",
      "mapping": {
        "#microsoft.graph.user": "#/components/schemas/microsoft.graph.user",
        "#microsoft.graph.group": "#/components/schemas/microsoft.graph.group"
      }
    }
  },
  "microsoft.graph.user": {
    "allOf": [
      { "$ref": "#/components/schemas/microsoft.graph.directoryObject" },
      {
        "title": "user",
        "type": "object"
      }
    ]
  },
  "microsoft.graph.group": {
    "allOf": [
      { "$ref": "#/components/schemas/microsoft.graph.directoryObject" },
      {
        "title": "group",
        "type": "object"
      }
    ]
  }
}
```

### Example 2 - using oneOf to constrain the derived types - not supported

```json
{
  "type": "object",
  "title": "directoryObject",
  "oneOf": [
    {
      "type": "object",
      "title": "user"
    },
    {
      "type": "object",
      "title": "group"
    }
  ]
}
```

## Default members

In addition to all the described properties, a model will contain a set of default members.

### Factory method

The factory method (static) is used by the parse node implementation to get the base or derived instance according to the discriminator value. If an operation describes returning a Person model, and the Person model has discriminator information (mapping + property name), and the response payload contains one of the mapped value (e.g. Employee), the deserialization process will deserialize the the derived Employee type instead of the base Person type. This way API client users can take advantage of the properties that are defined on this specialized model.

```csharp
public static new Person CreateFromDiscriminatorValue(IParseNode parseNode) {
    _ = parseNode ?? throw new ArgumentNullException(nameof(parseNode));
    var mappingValueNode = parseNode.GetChildNode("@odata.type");
    var mappingValue = mappingValueNode?.GetStringValue();
    return mappingValue switch {
        "#api.Employee" => new Employee(),
        _ => new Person(),
    };
}
```

### Field deserializers

The field deserializers method or property contains a list of callbacks to be used by the parse node implementation when deserializing the objects. Kiota relies on auto-serialization, where each type knows how to serialize/deserialize itself thanks to the OpenAPI description. A big advantage of this approach it to avoid tying the generated models to any specific serialization format (JSON, YAML, XML,...) or any specific library (because of attributes/annotations these libraries often require).

> [!NOTE]
> Any property found in the response payload and which does not match an entry in the field deserializers (casing included), will be stored in the additional data collection.

### Serialize method

Like the field deserializers, the model's serialize method leverages the passed serialization writer to serialize itself.

### Additional data

Dictionary/Map that stores all the additional properties which are not described in the schema.

> [!NOTE]
> The additional data property is only present when the OpenAPI description for the type allows it and if the current model doesn't inherit a model which already has this property.

### Backing store

When present, the property values are stored in this backing store instead of using fields for the object. The backing store allows multiple things like dirty tracking of changes, making it possible to get an object from the API, update a property, send that object back with only the changed property and not the full objects. Additionally it will be used for integration with third party data sources.

> [!NOTE]
> The backing store is only added if the target language supports it and when the `-b` parameter is passed to the CLI when generating the models.

### Properties and accessors name mangling

To produce a more idiomatic output for specific languages, mangling is applied to the properties names and/or accessors. The table below describes the mangling rules being applied for each language:

| Language   | Property name | Property accessors |
| ---------- | ------------- | ------------------ |
| CSharp     | `PascalCase`  | -                  |
| CLI        | `PascalCase`  | -                  |
| Go         | -             | `PascalCase`       |
| Java       | `camelCase`   | `camelCase`        |
| PHP        | -             | `camelCase`        |
| Python     | `snake_case`  | `snake_case`       |
| Ruby       | `snake_case`  | `snake_case`       |
| Swift      | -             | -                  |
| TypeScript | -             | `camelCase`        |

### Enumerations

Kiota will only project enum models when the schema type is **string** as projecting enums for boolean or number types does not add a lot of value.

Additionally, you can:

- Control the enum symbol name using the **x-ms-enum** extension. [more information](https://azure.github.io/autorest/extensions/#x-ms-enum)
- Control whether the projected enum should be flaggable through the **x-ms-enum-flags** extension. [more information](https://github.com/microsoft/OpenAPI/blob/main/extensions/x-ms-enum-flags.md)

During deserialization, if an unknown enum member value is encountered, the client will throw an exception. This design decision was made to ensure client applications fail as early as possible when the client might need to be refreshed or when the API description is inaccurate.
