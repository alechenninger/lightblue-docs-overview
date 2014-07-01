# Manual

## Overview

The purpose of this document is to give an overview of the lightblue
code structure and of some of the design decisions that may not be
obvious from the code alone.

### Core

The following components constitute the lightblue core:

* `util`: Contains the common utility classes and methods.

* `query-api`: Abstract syntax tree nodes for query, projection, update,
and sort expressions. These classes provide built-in support for
parsing an expression from a JSON document and constructing a JSON
document from an expression.

* `metadata`: Metadata definition classes and interface specification for
metadata manager. This project also contains the basic data types,
supported data constraints, and the basic metadata parsing framework.

* `crud`: Back-end independent parts of data management. Provides the
definitions for input/output interfaces, back-end CRUD implementation
interfaces, the mediator implementation, constraint checkers, query,
update and projection evaluators.

### Extensions

The core lightblue components don't provide any concrete
implementations or configuration facilities. Lightblue has to be
extended by:

* a configuration management layer that sets up and wires all the
required components together,
* a metadata implementation that manages metadata storage, and
* one or more CRUD implementations for specific back-ends

The components for managing these extensions are:

* `config`: provides an implementation of the configuration
management layer using JSON configuration files.

* `mongo/metadata`: provide the MongoDB implementation of metadata components.

* `mongo/crud`: provide the MongoDB implementation of CRUD components.

Once these extensions are in place, classes that configure lightblue
components and wire them together should be developed. The `config`
component provides one such implementation using JSON configuration
files.

## Metadata

Most of the classes in the metadata package represent the JSON
constructs in the [metadata
specification](https://github.com/bserdar/lightblue/wiki/Language-Spec-Metadata). The
two top-level classes are:

* `com.redhat.lightblue.metadata.Metadata`: The interface that
needs to be implemented by a metadata manager. The responsibilities of
this implementation is to store and maintain metadata information in
some sort of datastore. `mongo/metadata` project provides a metadata
implementation that stores entity metadata in a MongoDB collection.

* `com.redhat.lightblue.metadata.EntityMetadata`: This is the POJO that
describes the entity metadata. There are two sections:
    * EntityInfo: contains the part of the metadata that is not versioned.
    * EntitySchema: the part of the metadata that is versioned. The Metadata implementation
must deal with how to store the versions.

These metadata components can be extended by the concrete
implementation:
* types: The type system can be extended by providing new type
implementations that define what those types mean.
* constraints: There is a set of common constraints defined in
`com.redhat.lightblue.metadata.constraints` package, but more
constraints can be added by the implementation.
* hooks: There are no predefined hooks. All hooks are defined by the
particular implementation.
* data stores: The underlying data store containing the entity can be
defined here.

When defining these extensions, two extensions mechanisms are required:
1. a mechanism to construct Java objects representing these extensions (i.e. parser extensions)
1. a mechanism to interpret and execute the extension
For instance, constraints need a parser to parse constraint parameters, and a constraint checker that
actually checks the constraint. Similarly, data stores need a parser
to parse datastore configuration, and the actual datastore
implementation is a concrete implementation of the CRUDController
interface.

### Metadata Parser

Parser bits are under `com.redhat.lightblue.metadata.parser`
package. The main class is the MetadataParser class. This assumes that
the metadata is being parsed from some sort of document with an
abstract tree structure. This abstract tree has nodes that are values,
objects, or lists. Object nodes may have child nodes, and list nodes
may have indexed elements. The parser does not require any more
structure on the underlying document. The concrete implementations of
the parser should show the underlying metadata structure as a tree of
this format. A JSONMetadataParser is provided to parse
EntityMetadata objects from a JSON document and convert
EntityMetadata objects into a new JSON document.

All extensions except types are provided to the metadata parser by an
instance of Extensions class. Available types are given by an
implementation of TypeResolver interface.

### The Type System

Lightblue core provides the basic types that should be supported by
the concrete implementations and a way to extend the type system. A
data type is represented by the `Type` interface. Implementations for
the supported data types are under the
`com.redhat.lightblue.metadata.type` package.

The Type interface provides the `toJson` and `fromJson` methods to convert
values of that type to and from JSON, and the `cast()` method to convert
a value to a native Java object that represents that value. This can
be used to convert, for instance, a string value to an integer, or
vice versa.

Metadata parser uses an instance of `TypeResolver` to work with
types. It is expected by the application using Metadata to construct a
TypeResolver to resolve all supported types. The `Types` class is a
basic type resolver that also knows about the types already supported
by lightblue. That is:
```
import com.redhat.lightblue.metadata.Types;
...
   Types t=new Types();
   t.addDefaultTypeResolvers()
```
At this point, 't' is a type resolver that knows all the supported
types. If the application has support for additional types, it can
implement the Type interface, and extend the Types instance using that
type definition.

To implement new types:
* Define an implementation of Type interface. Define how that type can
be converted to/from JSON, and how that type is converted to/from Java
objects.
* Register the new type to the TypeResolver interface used in the
application. The TypeResolver implementation is passed to
MetadataParser and Mediator, so all the components become aware of the
new type.

### Constraints

The `com.redhat.lightblue.metadata.constraints` package defines the
basic entity and field level constraints. For each constraint defined
in this package, there is a parser in
`com.redhat.lightblue.metadata.parser` package, and a validator in
`com.redhat.lightblue.crud.validator` package. If additional constraints
are to be defined:
* Add a new constraint definition in metadata as an object of the form: `{ name: value }`
where name is the constraint name, and must be unique amongst all
constraints. The value can be a simple value, or a complicated object.
* Define a parser for the constraint that parses constraint
configuration. The parser is passed the constraint object, so it is up
to the parser to interpret the 'value'.
* Register that parser to
`com.redhat.lightblue.metadata.parser.Extensions` so whenever the
constraint is used in the metadata definition, the appropriate Java
objects are created and set in the EntityMetadata object.
* Implement a constraint checker for the constraint.
* Register the constraint checker to the instance of
com.redhat.lightblue.crud.Factory that is passed to Mediator. The same
Factory instance is used in the operations controlled by that mediator.

### Hooks

Hooks provide a way for CRUD operations to have side effects, similar to triggers in a relational database. This can for auditing
or for capturing certain changes and acting on them in some
way. Lightblue does not come with any predefined hooks. All hook
parsers must be registered with the `Extensions` instance to be used
by the metadata parser and actual hook implementations should be
registered with the `com.redhat.lightblue.crud.Factory` instance. The
hook parser should prepare a HookConfiguration object using the input.

### Using the metadata subsystem

The core metadata subsystem uses three lightblue packages: util,
query-api, and metadata. A concrete implementation of the metadata
subsytem could be hard-wired, or configuration-driven. For either case,
the following need to be done:
* Create new types if necessary for the implementation.
* Create an implementation of TypeResolver interface and register all
the types supported by the metadata implementation. The
`com.redhat.lightblue.metadata.types.DefaultTypes` class contains the
default types. If more or fewer types are needed, implement a custom resolver.
* Instantiate an Extensions instance.  See example below.
* Create new constraints if necessary and register all constraints with the
Extensions instance. Extensions provides mechanisms to add predefined
constraints.
* Create all hook parsers supported by the metadata implementations
and register them with the Extensions instance.
* If the underlying metadata is in JSON format, instantiate a
JSONMetadataParser. If not, create a new parser by sub-classing
MetadataParser and pass Extensions and the TypeResolver instance.
* Implement the Metadata interface, or if you're using MondoDB, use
`com.redhat.lightblue.metadata.mongo.MongoMetadata`. MongoMetadata gets a
DB, the Extensions<BSONObject> instance, and the TypeResolver, and
uses a BSONParser to parser metadata read from MongoDB.

#### Examples:

##### Create a metadata parser that works with JSON documents
```
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.node.JsonNodeFactory;
import com.redhat.lightblue.metadata.parser.Extensions;
import com.redhat.lightblue.metadata.parser.JSONMetadataParser;
import com.redhat.lightblue.metadata.mongo.MongoDataStoreParser;
import com.redhat.lightblue.metadata.types.DefaultTypes;
...
// Create extensions instance
Extensions<JsonNode> extensions = new Extensions<>();
// Add the default extensions (constraints)
extensions.addDefaultExtensions();
// Register a datastore parser for mongo data store
extensions.registerDataStoreParser("mongo", new
MongoDataStoreParser<JsonNode>());
// Create the parser instance with default types
JsonNodeFactory nodeFactory = JsonNodeFactory.withExactBigDecimals(true)
JSONMetadataParser parser = new JSONMetadataParser(extensions,
                                                   new DefaultTypes(),
                                                   nodeFactory);
```

##### Create a metadata parser that works with BSON documents
```
import org.bson.BSONObject;
import com.redhat.lightblue.metadata.parser.Extensions;
import com.redhat.lightblue.metadata.mongo.BSONParser;
import com.redhat.lightblue.metadata.mongo.MongoDataStoreParser;
import com.redhat.lightblue.metadata.types.DefaultTypes;
...
// Create extensions instance
Extensions<BSONObject> parserExtensions = new Extensions<>();
// Add the default extensions (constraints)
parserExtensions.addDefaultExtensions();
// Register a datastore parser for mongo data store
parserExtensions.registerDataStoreParser("mongo", new MongoDataStoreParser<BSONObject>());
DefaultTypes typeResolver = new DefaultTypes();
// Create the parser
BSONParser parser = new BSONParser(extensions, typeResolver);
```

### Mongo Metadata

The mongo/metadata project contains the MongoDB implementation of
Metadata interface. This project contains the following classes:

* BSONParser: Implementation of MetadataParser for BSON
  documents. When metadata is read from MongoDB or written into
  MongoDO collections, an instance of BSONParser is used.  BSONParser parsers BSON
  documents to EntityMetadata or converts EntityMetadata objects to
  BSON documents.
* MongoDataStoreParser: Parser for "mongo" data store implementation
* MongoDataStore: The "mongo" data store configuration
* MongoMetadata: The metadata implementation

EntityMetadata is saved as two separate documents into the same
"metadata" collection: EntityInfo and EntitySchema. EntityInfo is
saved with
```
 _id=<entityName> "|"
```
and EntitySchema is saved with
```
 _id=<entityName> "|" <schemaVersion>
```

This _id scheme keeps entity names and versions unique, and allows
quick access to entity metadata.

## CRUD

The crud/ contains these components:

* Mediator: This is front-end of the CRUD layer. All requests come to
  the mediator class and then routed to the relevant back-ends after
  validations. The mediator needs a Metadata implementation and an
  initialized Factory class. The configuration layer should construct
  these and create a Mediator instance to be used for all request
  processing.

* CRUD specification: Specifies the interfaces that should be
  implemented by CRUD controllers and the POJOs that are used to
  requests between the Mediator and CRUD controller implementations. A CRUD
  controller implementation performs the persistence operations on one entity using
  a particular back-end.

* Validators: The implementations for constraint validators.  In package `com.redhat.lightblue.crud.validator`.

* Evaluators: The query evaluator, document projector, and document updater classes.  In package `com.redhat.lightblue.eval`.

* Hooks: Defines the hook interfaces that
  should be implemented by the hook implementations, and the hook
  execution context that is used for hook execution.  In package `com.redhat.lightblue.hooks`.

The `com.redhat.lightblue` package contains the POJOs used for Mediator
input and output.

### Mediator

The `Mediator` class exposes the front-end APIs. It is constructed using
an instance of `Metadata` and an instance of
`com.redhat.lightblue.crud.Factory` classes. The Factory class is
similar to the Extensions class of metadata but it contains the
implementations for constraint checkers and the CRUD implementations.

#### Example Mediator instantiation supporting MongoDB
```
import org.bson.BSONObject;
import com.fasterxml.jackson.databind.node.JsonNodeFactory;
import com.mongodb.DB;
import com.redhat.lightblue.crud.Factory;
import com.redhat.lightblue.crud.validator.DefaultFieldConstraintValidators;
import com.redhat.lightblue.crud.mongo.MongoCRUDController;
import com.redhat.lightblue.crud.mongo.DBResolver;
import com.redhat.lightblue.mediator.Mediator;
import com.redhat.lightblue.metadata.types.DefaultTypes;
import com.redhat.lightblue.metadata.parser.Extensions;
import com.redhat.lightblue.metadata.mongo.MongoDataStore;
import com.redhat.lightblue.metadata.mongo.MongoDataStoreParser;
...
    // assumes a Mongo DB object is available.
    final DB db = ...;

    // create a DBResolver that wraps the DB object
    DBResolver simpleDBResolver = new DBResolver() {
        @Override
        public DB get(MongoDataStore s) {
            return db;
        }
    };
    // instantiate the Metadata
    Extensions<BSONObject> parserExtensions = new Extensions<>();
    parserExtensions.addDefaultExtensions();
    parserExtensions.registerDataStoreParser("mongo", new MongoDataStoreParser<BSONObject>());
    DefaultTypes typeResolver = new DefaultTypes();
    Metadata metadata = new MongoMetadata(db, parserExtensions, typeResolver);

    // Create factory instance
    Factory factory = new Factory();

    // Add default constraint validators to the factory
    factory.addFieldConstraintValidators(new DefaultFieldConstraintValidators());

    // Create a Mongo CRUD controller instance
    // MongoCRUDController uses a DBResolver instance to access the
    // MongoDB instances
    JsonNodeFactory nodeFactory = JsonNodeFactory.withExactBigDecimals(true);
    MongoCRUDController mongoCRUDController
              = new MongoCRUDController(nodeFactory, simpleDBResolver);

    // Register the CRUD controller with the factory
    factory.addCRUDController("mongo", mongoCRUDController);

    // Instantiate Mediator
    Mediator mediator=new Mediator(metadata, factory);
````

#### Operation Context

For each request, the Mediator creates an OperationContext
instance. The operation context (extending the CRUDOperationContext)
contains the following:

* All document contexts involved in the call.
    * insert/save: the documents provided by the input.
    * delete/update: the documents are initially empty, but then populated as documents are updated or
  deleted.
    * find: the list is initially empty, and later populated by the result set of the query.
* Each document context contains the following:
    * Original Document: The original version of the document, that is, the version of the document before any modifications are performed. This is null for insertions.
    * Updated Document: The dynamic copy of the document that all the operations performed on. At the end of a request, this contains the final copy of the document.
    * Projected Document: The output document, the projected version of the document that is included in the return
    * Document related errors.
* All metadata information for the entity involved in the
  operation. Every request specified an entity and version. Using that
  entity version, the operation context pre-loads all entities
  referenced by that request entity, and stored in the context. Any
  operation using the context need to specify the entity name only,
  the correct version is already loaded.
* Errors that are not related to any particular document.
* Caller's roles.
* Hook information.
* Status information.

### CRUD Controller

The `CRUDController` interface needs to be implemented by the
back-ends. Lightblue provides a MongoDB implementation of the
CRUDController: `com.redhat.lightblue.crud.mongo.MongoCRUDController`.

The CRUDController is expected to operate with the documents in
OperationContext. Before making any modifications to documents, the
implementation should store the original copy of every document in the
document context.

#### MongoCRUDController

This is the implementation of the CRUDController interface for
MongoDB. It is in `com.redhat.lightblue.crud.mongo` package. These are
the components of the implementation:

* Translator: This class translates query, sort, and update
  expressions to BSON expressions. It also translates JSON documents
  to BSON documents and vice versa. Not all update operations can be
  translated to BSON expressions. Therefore, whenever the translator
  detects an update operation that cannot be translated to BSON, it
  throws a CannotTranslateException and the CRUD implementation runs
  a read-update-write operation instead of using an atomic MongoDB update.
* DBResolver: This is an interface used by MongoCRUDController to get
  DB instances from MongoDataStore objects. The configuration layer
  should provide an implementation of the DBResolver class that
  retrieves a MongoDB DB object based on a give MongoDataStore
  configuration object.
* Operation implementations: These are the implementations of the
  DocSaver, DocUpdater, DocDeleter, DocFinder interfaces.
* MongoCRUDController: This is the CRUD controller implementation. The
  configuration layer should instantiate this class with an instance
  of DBResolver, and register it with the Factory instance before
  creating the Mediator.

### Evaluators

This is the `com.redhat.lightblue.eval` package. This package contains
the evaluators for QueryExpression, Projection, and UpdateExpression.

#### QueryEvaluator

A QueryEvaluator is constructed from a QueryExpression. It provides
the
```
    public abstract boolean evaluate(QueryEvaluationContext ctx);
```
method that returns if the document matches the query or not. The
QueryEvaluationContext contains the JSON document sub-tree at which the
query will be evaluated. After the evaluation, the query evaluation
context contains information about which elements of the arrays
matched the given criteria, and this information can be used by
Projection evaluators.

Lightblue MongoDB CRUD implementation does not use query evaluators to
search for documents. Any query expression is converted into a BSON
query and submitted to MongoDB for evaluation. However, projections
and update expressions may include nested queries, and these query
evaluators are used within those projections and update expressions.

#### Projector

Given a Projection expression, a Projector creates an evaluator of
that projection expression that can be used on multiple documents. A
projector provides the
```
    public abstract Boolean project(Path p, QueryEvaluationContext ctx);
````
method that returns whether a given path is included or excluded in
the projection. It returns null if the given path doesn't match any
part of the projection expression. This doesn't mean it is excluded,
though. If the path does not match any part of the projection
expression, but if the path points to an array or object, an element
or field of the array or object may match. The projector also provides
a method to project a document:
```
    public JsonDoc project(JsonDoc doc,
                           JsonNodeFactory factory,
                           QueryEvaluationContext ctx);
 ```
This method creates a new copy of the document, projected the
requested fields.

#### Updater

Given an update expression, Updater creates an Updater instance that
performs the update operation on a given document.
```
    public abstract boolean update(JsonDoc doc, FieldTreeNode contextMetadata, Path contextPath);
```
The update method performs the update operation on the given document,
starting from the contextPath node, and the corresponding metadata
node contextMetadata.

### Hooks

Hooks are queued as the operations are performed and then called once
the transaction is complete. This is done by the `HookManager`
class. The CRUDController implementation is expected to queue hooks
for execution and the mediator executes all hooks before
returning. HookManager keeps copies of the original and final copies
of the documents and passes them to the hook implementations when
they are called.
