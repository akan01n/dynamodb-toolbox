# DynamoDB Toolbox <!-- omit in toc -->

[![npm](https://img.shields.io/npm/v/dynamodb-toolbox.svg)](https://www.npmjs.com/package/dynamodb-toolbox)
[![npm](https://img.shields.io/npm/l/dynamodb-toolbox.svg)](https://www.npmjs.com/package/dynamodb-toolbox)
[![Coverage Status](https://coveralls.io/repos/github/jeremydaly/dynamodb-toolbox/badge.svg?branch=main)](https://coveralls.io/github/jeremydaly/dynamodb-toolbox?branch=main)
[![npm](https://img.shields.io/npm/dm/dynamodb-toolbox.svg)](https://www.npmjs.com/package/dynamodb-toolbox)

![dynamodb-toolbox](https://user-images.githubusercontent.com/2053544/69847647-b7910780-1245-11ea-8403-a35a0158f3aa.png)

## Single Table Designs have never been this easy! <!-- omit in toc -->

The **DynamoDB Toolbox** is a set of tools that makes it easy to work with [Amazon DynamoDB](https://aws.amazon.com/dynamodb/) and the [DocumentClient](https://docs.aws.amazon.com/sdk-for-javascript/v2/developer-guide/dynamodb-example-document-client.html). It's designed with **Single Tables** in mind, but works just as well with multiple tables. It lets you define your Entities (with typings and aliases) and map them to your DynamoDB tables. You can then **generate the API parameters** to `put`, `get`, `delete`, `update`, `query`, `scan`, `batchGet`, and `batchWrite` data by passing in JavaScript objects. The DynamoDB Toolbox will map aliases, validate and coerce types, and even write complex `UpdateExpression`s for you. 😉

## Why single table design?
Learn more about single table design in [Alex Debrie's blog](https://www.alexdebrie.com/posts/dynamodb-single-table).

## Version 0.8 🙌 <!-- omit in toc -->

Feedback is welcome and much appreciated! (Huge thanks to [@ThomasAribart](https://github.com/ThomasAribart) for all his work on this 🙌)

## Using AWS SDK v2?

dynamodb-toolbox@v0.8 and above is using AWS SDK v3.
If you are using AWS SDK v2, please use versions that are lower than 0.8.

## Docs & Community

- [Full Documentation](https://www.dynamodbtoolbox.com/docs)
- [GitHub Discussions](https://github.com/jeremydaly/dynamodb-toolbox/discussions)

## Quick Start

Using your favorite package manager, install DynamoDB Toolbox and the aws-sdk v3 dependencies in your project by running one of the following commands:

```bash
# npm
npm i dynamodb-toolbox
npm install @aws-sdk/lib-dynamodb @aws-sdk/client-dynamodb

# yarn
yarn add dynamodb-toolbox
yarn add @aws-sdk/lib-dynamodb @aws-sdk/client-dynamodb
```

Require or import `Table` and `Entity` from `dynamodb-toolbox`:

```typescript
import { Table, Entity } from 'dynamodb-toolbox'
```

Create a Table (with the DocumentClient using aws-sdk v3):

```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb'
import { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'

const marshallOptions = {
  // Whether to automatically convert empty strings, blobs, and sets to `null`.
  convertEmptyValues: false, // if not false explicitly, we set it to true.
  // Whether to remove undefined values while marshalling.
  removeUndefinedValues: false, // false, by default.
  // Whether to convert typeof object to map attribute.
  convertClassInstanceToMap: false, // false, by default.
}

const unmarshallOptions = {
  // Whether to return numbers as a string instead of converting them to native JavaScript numbers.
  // NOTE: this is required to be true in order to use the bigint data type.
  wrapNumbers: false, // false, by default.
}

const translateConfig = { marshallOptions, unmarshallOptions }

// Instantiate a DocumentClient
export const DocumentClient = DynamoDBDocumentClient.from(new DynamoDBClient({}), translateConfig)


// Instantiate a table
const MyTable = new Table({
  // Specify table name (used by DynamoDB)
  name: 'my-table',

  // Define partition and sort keys
  partitionKey: 'pk',
  sortKey: 'sk',

  // Add the DocumentClient
  DocumentClient
})
```

Create an Entity:

```typescript
const Customer = new Entity({
  // Specify entity name
  name: 'Customer',

  // Define attributes
  attributes: {
    id: { partitionKey: true }, // flag as partitionKey
    sk: { hidden: true, sortKey: true }, // flag as sortKey and mark hidden
    age: { type: 'number' }, // set the attribute type
    name: { type: 'string', map: 'data' }, // map 'name' to table attribute 'data'
    emailVerified: { type: 'boolean', required: true }, // specify attribute as required
    co: { alias: 'company' }, // alias table attribute 'co' to 'company'
    status: ['sk', 0], // composite key mapping
    date_added: ['sk', 1] // composite key mapping
  },

  // Assign it to our table
  table: MyTable

  // In Typescript, the "as const" statement is needed for type inference
} as const)
```

Put an item:

```typescript
// Create an item (using table attribute names or aliases)
const customer = {
  id: 123,
  age: 35,
  name: 'Jane Smith',
  emailVerified: true,
  company: 'ACME',
  status: 'active',
  date_added: '2020-04-24'
}

// Use the 'put' method of Customer:
await Customer.put(customer)
```

The item will be saved to DynamoDB like this:

```typescript
{
  "pk": 123,
  "sk": "active#2020-04-24",
  "age": 35,
  "data": "Jane Smith",
  "emailVerified": true,
  "co": "ACME",
  // Attributes auto-generated by DynamoDB-Toolbox
  "_et": "customer", // Entity name (required for parsing)
  "_ct": "2021-01-01T00:00:00.000Z", // Item creation date (optional)
  "_md": "2021-01-01T00:00:00.000Z" // Item last modification date (optional)
}
```

You can then get the data:

```typescript
// Specify primary key
const primaryKey = {
  id: 123,
  status: 'active',
  date_added: '2020-04-24'
}

// Use the 'get' method of Customer
const response = await Customer.get(primaryKey)
```

Since v0.4, the method inputs, options and response types are inferred from the Entity definition:

```typescript
await Customer.put({
  id: 123,
  // ❌ Sort key is required ("sk" or both "status" and "date_added")
  age: 35,
  name: ['Jane', 'Smith'], // ❌ name should be a string
  emailVerified: undefined, // ❌ attribute is marked as required
  company: 'ACME'
})

const { Item: customer } = await Customer.get({
  id: 123,
  status: 'active',
  date_added: '2020-04-24' // ✅ Valid primary key
})
type Customer = typeof customer
// 🙌 Type is equal to:
type ExpectedCustomer =
  | {
      id: any
      age?: number | undefined
      name?: string | undefined
      emailVerified: boolean
      company?: any
      status: any
      date_added: any
      entity: string
      created: string
      modified: string
    }
  | undefined
```

See [Type Inference](https://www.dynamodbtoolbox.com/docs/type-inference) in the documentation for more details.

## Features

- **Table Schemas and DynamoDB Typings:** Define your Table and Entity data models using a simple JavaScript object structure, assign DynamoDB data types, and optionally set defaults.
- **Magic UpdateExpressions:** Writing complex `UpdateExpression` strings is a major pain, especially if the input data changes the underlying clauses or requires dynamic (or nested) attributes. This library handles everything from simple `SET` clauses, to complex `list` and `set` manipulations, to defaulting values with smartly applied `if_not_exists()` to avoid overwriting data.
- **Bidirectional Mapping and Aliasing:** When building a single table design, you can define multiple entities that map to the same table. Each entity can reuse fields (like `pk` and`sk`) and map them to different aliases depending on the item type. Your data is automatically mapped correctly when reading and writing data.
- **Composite Key Generation and Field Mapping:** Doing some fancy data modeling with composite keys? Like setting your `sortKey` to `[country]#[region]#[state]#[county]#[city]#[neighborhood]` model hierarchies? DynamoDB Toolbox lets you map data to these composite keys which will both autogenerate the value _and_ parse them into fields for you.
- **Type Coercion and Validation:** Automatically coerce values to strings, numbers and booleans to ensure consistent data types in your DynamoDB tables. Validate `list`, `map`, and `set` types against your data. Oh yeah, and `set`s are automatically handled for you. 😉
- **Powerful Query Builder:** Specify a `partitionKey`, and then easily configure your sortKey conditions, filters, and attribute projections to query your primary or secondary indexes. This library can even handle pagination with a simple `.next()` method.
- **Simple Table Scans:** Scan through your table or secondary indexes and add filters, projections, parallel scans and more. And don't forget the pagination support with `.next()`.
- **Filter and Condition Expression Builder:** Build complex Filter and Condition expressions using a standardized `array` and `object` notation. No more appending strings!
- **Projection Builder:** Specify which attributes and paths should be returned for each entity type, and automatically filter the results.
- **Secondary Index Support:** Map your secondary indexes (GSIs and LSIs) to your table, and dynamically link your entity attributes.
- **Batch Operations:** Full support for batch operations with a simpler interface to work with multiple entities and tables.
- **Transactions:** Full support for transaction with a simpler interface to work with multiple entities and tables.
- **Default Value Dependency Graphs:** Create dynamic attribute defaults by chaining other dynamic attribute defaults together.
- **TypeScript Support:** v0.4 of this library provides strong typing support AND type inference 😍. Inferred type can still overriden with [Overlays](#overlays). Some [Utility Types](#utility-types) are also exposed. Additional work is still required to support schema validation & typings.

## Additional References

- [DocumentClient SDK Reference](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/modules/_aws_sdk_lib_dynamodb.html)
- [Best Practices for DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [DynamoDB, explained.](https://www.dynamodbguide.com/)
- [The DynamoDB Book](https://www.dynamodbbook.com/)

## Contributions and Feedback

Contributions, ideas and bug reports are welcome and greatly appreciated. Please add [issues](https://github.com/jeremydaly/dynamodb-toolbox/issues) for suggestions and bug reports or create a pull request. You can also contact me on Twitter: [@jeremy_daly](https://twitter.com/jeremy_daly).
