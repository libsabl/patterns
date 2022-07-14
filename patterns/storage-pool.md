sabl / [patterns](../README.md#patterns) / storage pool

# storage pool
 
<small>depends on: [**context**](./context.md)</small>

**storage-pool** is a simple, [context](./context.md)-aware pattern for describing **connection pooling** and **storage transactions** agnostic of the underlying storage type. This same pattern works for relational, document, key-value, graph, and other storage architectures. 

Defining these interfaces directly allows authors to write effective business logic that includes basic CRUD actions and even transaction workflows, without depending on a specific storage type, let alone a specific proprietary driver or client library. This is in turn allows **concise and testable code** while avoiding over-dependence on implementation details of underlying storage choices.

The design of the top-level storage interfaces as well as the derived relational interfaces is based on the go standard library [`database/sql` package](https://pkg.go.dev/database/sql). That package actually does not define explicit interfaces for the pool (`DB`), connection (`Conn`), or transaction (`Txn`) types. This work extracts the common patterns implicit in that package.

## Why Bother?

There is clear value in abstracting the common operations of a specific storage type, such as a relational database. Setting aside variations in, for example, SQL dialects, it is still helpful to have a uniform view of interacting with a relational database without having to revisit or rewrite higher-level code if one switches from MySQL to SQL Server. This was surely the motivation behind the original go [`database/sql` package](https://pkg.go.dev/database/sql) which established a common API for all relational database clients, or the [System.Data.Common](https://docs.microsoft.com/en-us/dotnet/api/system.data.common) APIs that provide the same uniformity in the dotnet ecosystem.

But what about the higher level presented here: the idea of a storage pool or even a transaction without even knowing whether it is a relational database, a document store, or something else?

The answer here is [also addressed below](#agnostic-of-storage-type-specific-to-lifecycle) but the essence is that storage-type-agnostic transactions are useful in the business logic layers of a well-structured application:

Business logic **does not need to know the underlying storage type** even to invoke abstract CRUD operations on known record types. "Store an invoice" can be described in APIs that do not change whether the invoice's data is inserted in a table of a relational database, appended or merged into a document store, distributed in fragments across a wide-column store, or just added to an in-memory object graph that allows full fidelity testing of higher-level business logic without dependence on an external storage service.

Business logic **does** care about expressing when a series of operations should all succeed or fail together. This is the essence of the transaction concept. Ideally transactions can be pushed all the way down to the storage layer (`START TRANSACTION / COMMIT`), while others may be effectively implemented on the client, at least partially, by queuing the execution of a series of storage operations until the conceptual transaction is committed. Either way it is a human concept that very much belongs in the business logic layers to decide whether it is important to link the fate of a series of specific storage operations.

Drawing hard lines between layers, across which knowledge and information may not cross, is an essential activity in designing and building robust and maintainable software. Likewise, describing limited interfaces that can be easily mocked or replaced for testing is essential for building digital machines that can be tested well: where high code coverage is achievable, and high code coverage naturally achieves high *behavior* coverage. Providing for transactions while prohibiting business logic from even knowing the storage type is a powerful tool for furthering these engineering goals.

## Type-Specific Storage Operations

The highest level of the storage pattern is agnostic of storage type, but includes no actual direct storage operations. Instead, each storage type (relational, graph, document, etc) tends to support a set of operations that are specific to that storage type but are supported by all platforms of that type. Relational databases can all accept a string SQL statement and list of parameters and either `exec` it for simple row count or `query` it for a scrollable cursor. Graph databases support the Gremlin API. Document stores all provide operations such as `insertOne` or `find`. Key-value stores all support basic `set`, `get`, and `del` operations, often with variants that control expiration and eviction.

Consequently each storage type can often declare its own more derived interfaces that represent a union of the top-level pool, connection, and transaction interfaces with the type-specific storage operations. See [below](#type-specific-apis-example-stack) for an illustration of this.

<!--  
The sabl project has defined the following type-specific patterns:

- [Storage Pool: Relational](./storage-relational.md)

-->

## Implementations
  
Implementations exist at three levels:

1. **Top-level abstract (storage pool)**

   These libraries contain only the interfaces of the top-level storage pool pattern and an implementation of a storage-agnostic transaction workflow. The patterns implemented in these libraries are described in this document.

2. **Type-specific abstract**

   These libraries describe the common CRUD APIs for a specific storage type, such as standard SQL relational databases. They include interfaces that are a composition of the base storage pool interfaces with the CRUD APIs (e.g. `exec` and `query`) specific to the storage type. The patterns implemented in these libraries are described in the [storage-type-specific patterns](#type-specific-storage-operations) noted above.

3. **Platform adapters**

   These libraries wrap the proprietary client API of a particular storage platform to expose them as the common interfaces for the applicable storage type. 

The table below lists all registered implementations for all levels, types, and platforms:
 
|Language|Type|Platform|Maintainer|Source|Package|
|-|-|-|-|-|-|
|JS / TS|all|all|sabl|github : [libsabl/storage-pool-js](https://github.com/libsabl/storage-pool-js)|[@sabl/storage-pool](https://www.npmjs.com/package/@sabl/storage-pool)|
|JS / TS|relational|all|sabl|github : [libsabl/rdb-api-js](https://github.com/libsabl/rdb-js)|[@sabl/rdb-api](https://www.npmjs.com/package/@sabl/rdb-api)|
|JS / TS|relational|[MySQL](https://www.mysql.com)|sabl|github : [libsabl/rdb-mysql-js](https://github.com/libsabl/rdb-mysql-js)|[@sabl/rdb-mysql](https://www.npmjs.com/package/@sabl/rdb-mysql)|
 
## Basic pattern - Abstract Storage Pool
 
The entire top-level storage pool pattern consists of three related concepts: A **pool**, a **connection**, and a **transaction**. 
 
In any given storage type, all three of these interfaces implement all of the CRUD APIs for the applicable type. That is, a relational SQL `query` could be invoked on a pool, a connection, or a transaction. The difference is in session continuity and in the effect of canceling the context provided to the operation:

**Pool**
  - **Lifecycle**: A connection is retrieved from the pool, and is [closed](#note-on-terminology--closed-vs-destroyed) and returned to the pool as soon as the single operation is complete.

  - **On Completion**: If the operation completed successfully, it is immediately committed (usually implicitly by the storage service), and the connection is automatically [closed](#note-on-terminology--closed-vs-destroyed) and returned to the pool.
  
  - **On Cancellation**
    - If **a connection is not yet available**, the operation is terminated immediately and the request to the pool is aborted
    - If a connection was obtained and the operation started, the operation is aborted, and the connection is [closed](#note-on-terminology--closed-vs-destroyed) and returned to the pool

**Connection**
  - **Lifecycle**: The underlying connection remains open until it is canceled or explicitly closed. This allows maintaining session state such as variables, settings, temporary tables, or open transactions, at the cost of holding on to a connection

  - **On Completion**: If the operation completed successfully, it is immediately committed (usually implicitly by the storage service), but the connection remains open and is not returned to the pool.
  
  - **On Cancellation**:
    - **If a non-transaction operation is in progress**, it is immediately aborted and the connection is [closed](#note-on-terminology--closed-vs-destroyed) and returned to the pool.
    - **If a transaction is open**, it is rolled back, and the connection is [closed](#note-on-terminology--closed-vs-destroyed) and returned to the pool.
    - **If no transaction is open, and no operation is in progress**, the connection is simply [closed](#note-on-terminology--closed-vs-destroyed) and returned to the pool with aborting any existing operations.

**Transaction**
  - **Lifecycle**: The underlying transaction remains open and uncommitted until the transaction is canceled or is explicitly committed or rolled back. When the transaction is committed or rolled back the underlying connection remains open, though the connection would also be closed automatically if the transaction was created directly from a pool.
  
  - **On Completion**: A transaction is only completed by explicitly committing it. Some patterns may call the commit API on the transaction on the successful completion of a callback. 

  - **On Cancellation**:
    - Any ongoing operation is immediately aborted, and the transaction is rolled back. The underlying connection remains open, though the connection would also be closed automatically if the transaction was created directly from a pool.
  
### Note on terminology : "closed" vs "destroyed"

The purpose of connection pooling is to keep negotiated, authenticated connections to a remote storage service open and available so that they can be quickly obtained and used by a client application. At the same time, it desirable to prevent unintended side effects where one operation on a pooled connection affects subsequent but logically unrelated operations executed on the same connection.  

To represent this, both APIs and this documentation use the term "closed" to mean marking the connection as exposed to client code as no longer active for subsequent operations, but without actually closing the underlying protocol connection. The term "destroy" is used to mean actually closing the underlying protocol connection and entirely removing it from the pool.

The storage pattern described here intentionally does not include a destroy method on individual connections. The point of using a pool is to delegate responsibility for initializing and destroying protocol connections to the implementation details of a pool.

### Abstract APIs

The entire structural API can be summarized by the following interface definition, written here in TypeScript merely for example purposes. Details of each method are described below. Again, note that for any given storage type, the pool, connection, and transaction interfaces will *also* implement type-specific data operations such as `query` for a relational database, `insertOne` for a document store, or `set` for a key-value store.

```ts
interface StoragePool {
  conn(ctx: Context): Promise<StorageConn>;
  beginTxn(ctx: IContext, opts?: TxnOptions): Promise<StorageTxn>;
  close(): Promise<void>;
}

interface StorageConn {
  beginTxn(ctx: IContext, opts?: TxnOptions): Promise<StorageTxn>;
  close(): Promise<void>;
}

interface StorageTxn {
  commit(): Promise<void>;
  rollback(): Promise<void>;
}

// Transaction options that may or may not be supported
// by a particular storage type or platform
interface TxnOptions {
  readonly isolationLevel?: IsolationLevel;
  readonly readOnly?: boolean;
}
 
enum IsolationLevel {
  default,
  readUncommitted,
  readCommitted,
  writeCommitted,
  repeatableRead,
  snapshot,
  serializable,
  linearizable,
}
```

## Type-Specific APIs Example: Stack

As noted above, for any particular storage type, such as a relational database, there are also type-specific APIs that are defined and which are available on all three basic interface types (pool, connection, transaction).

Imagine for example a storage type that is simply a stack which supports `push`, `peek`, and `pop` of a single item. The common type-specific operations could be described as follows:

**Stack operations API: TypeScript**
```ts
export interface StackOps {
  push(ctx: Context, val: any): Promise<void>;
  peek(ctx: Context): Promise<any>;
  pop(ctx: Context): Promise<any>;
}
```

**Stack operations API: Go**
```go
type StackOps interface {
    Push(ctx context.Context, val interface{}) error
    Peek(ctx context.Context) (interface{}, error)
    Pop(ctx context.Context) (interface{}, error)
}
```

**Stack operations API: C#**
```cs
public interface StackOps {
    Task Push(IContext ctx, object value);
    Task<object> Peek(IContext ctx);
    Task<object> Pop(IContext ctx);
}
```

The composed pool, connection, and transaction interfaces would then be as follows. Note that the `Pool.conn` and the `beginTxn` APIs are generally shadowed or overridden to return the type-specific derived connection or transaction type, rather than the generic `StorageConn` and `StorageTxn`.

**Full Stack API: TypeScript**

```ts
/* Full APIs shown for clarity. Real code would use `extends` to mix-in StackOps */

// Structurally implements StackOps and StoragePool
export interface StackPool {
  push(ctx: Context, val: any): Promise<void>;
  peek(ctx: Context): Promise<any>;
  pop(ctx: Context): Promise<any>;

  conn(ctx: Context): Promise<StackConn>;
  beginTxn(ctx: IContext, opts?: TxnOptions): Promise<StackTxn>;
  close(): Promise<void>;
}

// Structurally implements StackOps and StorageConn
export interface StackConn {
  push(ctx: Context, val: any): Promise<void>;
  peek(ctx: Context): Promise<any>;
  pop(ctx: Context): Promise<any>;

  beginTxn(ctx: IContext, opts?: TxnOptions): Promise<StackTxn>;
  close(): Promise<void>;
}

// Structurally implements StackOps and StorageConn
export interface StackTxn {
  push(ctx: Context, val: any): Promise<void>;
  peek(ctx: Context): Promise<any>;
  pop(ctx: Context): Promise<any>;

  beginTxn(ctx: IContext, opts?: TxnOptions): Promise<StackTxn>;
  close(): Promise<void>;
}
```

The key result is two complementary views of the same objects:

#### **Agnostic of lifecycle, specific to storage type**

In the lower levels of a data access layer or equivalent, it is necessary to know the actual storage type while at the same time not caring if the current context is an ephemeral pool operation, a long-lived connection, or an open transaction. If the current concern is to "create an invoice record in the actual data store", the relational `INSERT` or document `insertOne` operation is identical whether or not the context is a pool, connection or transaction.

In our stack example, these locations would treat all three interfaces simply as a `StackOps`.

#### **Agnostic of storage type, specific to lifecycle**

In higher levels of business logic where abstract CRUD operations are validated and composed, it is common to need to express the concept of a transaction (all these things must succeed or fail together), without knowing or caring whether the underlying storage type is relation, document, or otherwise. The key concept is: create the invoice, and create the first line, and create the receipt, and create the audit log, and update the customer summary, and either commit it all together or rollback partial work. 

In these places, code will treat the interfaces as abstract `StoragePool`, `StorageConn`, and `StorageTxn` interfaces. In fact, it is possible to implement a generic transaction workflow exclusively with these top-level interfaces, and sabl implementations of the top-level interfaces provide this.

---
&copy; 2022 Joshua Honig. All rights reserved. See [LICENSE](../LICENSE.md).