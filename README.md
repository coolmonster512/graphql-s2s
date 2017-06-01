<a href="https://neap.co" target="_blank"><img src="https://neap.co/img/neap_black_small_logo.png" alt="Neap Pty Ltd logo" title="Neap" align="right" height="50" width="120"/></a>


# GraghQL Schema-2-Schema Transpiler
[![NPM][1]][2] [![Tests][3]][4]

[1]: https://img.shields.io/npm/v/graphql-s2s.svg?style=flat
[2]: https://www.npmjs.com/package/graphql-s2s
[3]: https://travis-ci.org/nicolasdao/graphql-s2s.svg?branch=master
[4]: https://travis-ci.org/nicolasdao/graphql-s2s

## What It Does
Defining GraphQL schemas that could look like this:
```js
const schema = `
# Defining a custom 'node' metadata attribute
@node
type Node {
	id: ID!
}

# Inheriting from the 'Node' type
type Person inherits Node {
	firstname: String
	lastname: String
}

type Student inherits Person {
	nickname: String

	# Defining another custom 'edge' metadata, and supporting a generic type
	@edge(some other metadata using whatever syntax I want)
	questions: Paged<Question>
}

type Question inherits Node {
	name: String!
	text: String!
}

# Defining a generic type
type Paged<T> {
	data: [T]
	cursor: ID
}
`
```
GraphQL S2S allows to enrich the standard GraphQL Schema string used by both [graphql.js](https://github.com/graphql/graphql-js) and the [Apollo Server](https://github.com/apollographql/graphql-tools). The enriched schema supports:
- [**Type Inheritance**](#type-inheritance)
- [**Generic Types**](#generic-types)
- [**Metadata Decoration**](#metadata-decoration)

The enriched schema provides a richer and more compact notation. The transpiler converts the enriched schema into the standard expected by [graphql.js](https://github.com/graphql/graphql-js) (using the _buildSchema_ method) as well as the [Apollo Server](https://github.com/apollographql/graphql-tools).

_Metadata_ can be added to decorate the schema types and properties. Add whatever you want as long as it starts with _@_ and start hacking your schema. The original intent of that feature was to decoration the schema with metadata _@node_ and _@edge_ so we coould add metadata about the nature of the relations between types.

## Install
```js
npm install 'graphql-s2s' --save
```

## Examples
_WARNING: the following examples will be based on '[graphql-tools](https://github.com/apollographql/graphql-tools)' from the Apollo team, but the string schema could also be used with the 'buildSchema' method from graphql.js_

### Type Inheritance
_NOTE: The examples below only use 'type', but it would also work on 'input'_

__*Before graphql-s2s*__
```js
const schema = `
type Teacher {
	id: ID!
	creationDate: String

	firstname: String!
	middlename: String
	lastname: String!
	age: Int!
	gender: String 

	title: String!
}

type Student {
	id: ID!
	creationDate: String

	firstname: String!
	middlename: String
	lastname: String!
	age: Int!
	gender: String 

	nickname: String!
}`

```
__*After graphql-s2s*__
```js
const schema = `
type Node {
	id: ID!
	creationDate: String
}

type Person inherits Node {
	firstname: String!
	middlename: String
	lastname: String!
	age: Int!
	gender: String 
}

type Teacher inherits Person {
	title: String!
}

type Student inherits Person {
	nickname: String!
}`

```

__*Full code example*__

```js
const { transpileSchema } = require('graphql-s2s');
const { makeExecutableSchema } = require('graphql-tools');
const { students, teachers } = require('./dummydata.json');

const schema = `
type Node {
	id: ID!
	creationDate: String
}

type Person inherits Node {
	firstname: String!
	middlename: String
	lastname: String!
	age: Int!
	gender: String 
}

type Teacher inherits Person {
	title: String!
}

type Student inherits Person {
	nickname: String!
	questions: [Question]
}

type Question inherits Node {
	name: String!
	text: String!
}

type Query {
  # ### GET all users
  #
  students: [Student]

  # ### GET all teachers
  #
  teachers: [Teacher]
}
`

const resolver = {
        Query: {
            students(root, args, context) {
                return Promise.resolve(students);
            },

            teachers(root, args, context) {
                return Promise.resolve(teachers);
            }
        }
    };

const executableSchema = makeExecutableSchema({
  typeDefs: [transpileSchema(schema)],
  resolvers: resolver
});
```

### Generic Types
_NOTE: The examples below only use 'type', but it would also work on 'input'_

__*Before graphql-s2s*__
```js
const schema = `
type Teacher {
	id: ID!
	creationDate: String
	firstname: String!
	middlename: String
	lastname: String!
	age: Int!
	gender: String 
	title: String!
}

type Student {
	id: ID!
	creationDate: String
	firstname: String!
	middlename: String
	lastname: String!
	age: Int!
	gender: String 
	nickname: String!
	questions: Questions
}

type Question {
	id: ID!
	creationDate: String
	name: String!
	text: String!
}

type Teachers {
	data: [Teacher]
	cursor: ID
}

type Students {
	data: [Student]
	cursor: ID
}

type Questions {
	data: [Question]
	cursor: ID
}

type Query {
  # ### GET all users
  #
  students: Students

  # ### GET all teachers
  #
  teachers: Teachers
}
`

```
__*After graphql-s2s*__
```js
const schema = `
type Paged<T> {
	data: [T]
	cursor: ID
}

type Node {
	id: ID!
	creationDate: String
}

type Person inherits Node {
	firstname: String!
	middlename: String
	lastname: String!
	age: Int!
	gender: String 
}

type Teacher inherits Person {
	title: String!
}

type Student inherits Person {
	nickname: String!
	questions: Paged<Question>
}

type Question inherits Node {
	name: String!
	text: String!
}

type Query {
  # ### GET all users
  #
  students: Paged<Student>

  # ### GET all teachers
  #
  teachers: Paged<Teacher>
}
`
```
This is very similar to C# or Java generic classes. What the transpiler will do is to simply recreate 3 types (one for Paged\<Question\>, Paged\<Student\> and Paged\<Teacher\>). If we take the Paged\<Question\> example, the transpiled type will be:
```js
type PagedQuestion {
	data: [Question]
	cursor: ID
}
```

__*Full code example*__

```js
const { transpileSchema } = require('graphql-s2s');
const { makeExecutableSchema } = require('graphql-tools');
const { students, teachers } = require('./dummydata.json');

const schema = `
type Paged<T> {
	data: [T]
	cursor: ID
}

type Node {
	id: ID!
	creationDate: String
}

type Person inherits Node {
	firstname: String!
	middlename: String
	lastname: String!
	age: Int!
	gender: String 
}

type Teacher inherits Person {
	title: String!
}

type Student inherits Person {
	nickname: String!
	questions: Paged<Question>
}

type Question inherits Node {
	name: String!
	text: String!
}

type Query {
  # ### GET all users
  #
  students: Paged<Student>

  # ### GET all teachers
  #
  teachers: Paged<Teacher>
}
`

const resolver = {
        Query: {
            students(root, args, context) {
                return Promise.resolve({ data: students.map(s => ({ __proto__:s, questions: { data: s.questions, cursor: null }})), cursor: null });
            },

            teachers(root, args, context) {
                return Promise.resolve({ data: teachers, cursor: null });
            }
        }
    };

const executableSchema = makeExecutableSchema({
  typeDefs: [transpileSchema(schema)],
  resolvers: resolver
});
```

### Metadata Decoration
Define your own custom metadata and decorate your GraphQL schema with new types of data. Let's imagine we want to explicitely add metadata about the type of relations between nodes, we could write something like this:
```js
const { getSchemaParts } = require('graphql-s2s');
const schema = `
@node
type User {
	@edge('<-[CREATEDBY]-')
	posts: [Post]
}
`

const schemaObjects = getSchemaParts(schema);

// -> schemaObjects
//
// [ { type: 'TYPE',
//     name: 'User',
//     metadata:
//      { name: 'node',
//        body: '',
//        schemaType: 'TYPE',
//        schemaName: 'User',
//        parent: null },
//     genericType: null,
//     blockProps: [ { 	comments: '',
//					    details: {
//							name: 'posts',
//					       	metadata: { 
//								name: 'edge',
//								body: '(\'<-[CREATEDBY]-\')',
//								schemaType: 'PROPERTY',
//								schemaName: 'posts: [Post]',
//								parent: { 
//									type: 'TYPE',
//									name: 'User',
//									metadata: { type: 'TYPE', name: 'node' } } },
//					       	params: null,
//					       	result: { originName: '[Post]', isGen: false, name: '[Post]' } },
//					    value: 'posts: [Post]' } ],
//     inherits: null,
//     implements: null,
//     comments: undefined } ]
```


## This Is What We re Up To
We are Neap, an Australian Technology consultancy powering the startup ecosystem in Sydney. We simply love building Tech and also meeting new people, so don't hesitate to connect with us at [https://neap.co](https://neap.co).