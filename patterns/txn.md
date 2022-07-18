sabl / [patterns](../README.md#patterns) / txn

# txn
 
<small>depends on: [**context**](./context.md)</small>

**txn** is a simple, [context](https://github.com/libsabl/patterns/blob/main/patterns/context.md)-aware pattern for describing transactions: batches of operations which should all succeed together or be rolled back. The pattern can be used to run actual storage system transactions, but it is also useful for running conceptual transactions purely in a client runtime, which avoid the blocking costs of native database transactions but still allow clean up of resources if a series operations does not all succeed. 

The design of the top-level interfaces is based on the transaction APIs in the go standard library [`database/sql` package](https://pkg.go.dev/database/sql).

### Motivation

Although transactions are often implemented at the lowest levels of a physical storage layer, human decisions about when to use them are often made at much higher levels of business logic. This is because considerations about which records or "related" and "should" succeed of fail together tend to be about the human meaning of the records as much as about their mechanical shape.

In well-structured code bases the decisions about which operations to link together in a transaction are often implemented in layers that should not even know about the specifics of storage. They simply need to "store an Invoice" or "delete a Product", they should not care if those operations are implemented in SQL statements, operations on a wide-column store, or in-memory operations for a mock repository provided during testing.

Defining these interfaces and algorithms in the abstract allows authors to write effective business logic that includes transaction workflows, without depending on a specific storage type, let alone a specific proprietary driver. This is in turn allows concise and testable code while avoiding over-dependence on implementation details of underlying storage choices.
 
## Implementations
  
Implementations exist at three levels:

1. **Top-level abstract (txn)**

   These libraries contain the interfaces of the top-level txn pattern, an implementation of ChangeSet, and an implementation of storage-agnostic transaction workflows. The patterns implemented in these libraries are described in this document.

2. **Type-specific abstract**

   These libraries describe the common CRUD APIs for a specific storage type, such as standard SQL relational databases. They include interfaces that are a composition of the base `Txn` and `Transactable` interfaces with the CRUD APIs (e.g. `exec` and `query`) specific to the storage type.

3. **Platform adapters**

   These libraries wrap the proprietary client API of a particular storage platform to expose them as the common interfaces for the applicable storage type.

The table below lists all registered implementations for all levels, types, and platforms:
 
|Language|Type|Platform|Maintainer|Source|Package|
|-|-|-|-|-|-|
|JS / TS|all|all|sabl|github : [libsabl/txn-js](https://github.com/libsabl/txn-js)|[@sabl/txn](https://www.npmjs.com/package/@sabl/txn)|
|JS / TS|relational|all|sabl|github : [libsabl/rdb-api-js](https://github.com/libsabl/rdb-js)|[@sabl/rdb-api](https://www.npmjs.com/package/@sabl/rdb-api)|
|JS / TS|relational|[MySQL](https://www.mysql.com)|sabl|github : [libsabl/rdb-mysql-js](https://github.com/libsabl/rdb-mysql-js)|[@sabl/rdb-mysql](https://www.npmjs.com/package/@sabl/rdb-mysql)|
 
## Basic Pattern

The basic pattern is represented in the complementary `Txn` and `Transactable` interfaces. [Transaction workflows](#runners) can be implemented with little more than these two interfaces.

### `Txn` interface

A `Txn` is a minimal interface that represents a set of pending operations can be either committed or rolled back. Both operations accept a context, which may or may not be used by an underlying implementation.

**Txn: TypeScript**

```ts
interface Txn {
  commit(ctx: IContext): Promise<void>;
  rollback(ctx: IContext): Promise<void>;
}
```
 
**Txn: Go**

```go
// Respect existing "Tx" naming convention
type Tx interface {   
  Commit(ctx context.Context) error
  Rollback(ctx context.Context) error
}
```

**Txn: C#**
```ts
public interface Txn {
  Task Commit(IContext ctx);
  Task Rollback(IContext ctx);
}
```


### `Transactable` interface

A transactable is any object that can start a transaction. Often this is a database pool or connection. If an underlying service supports nested transactions, the transactable could itself be a [transaction](#txn-interface).

**Transactable: TypeScript**
```ts
interface Transactable<T extends Txn> {
  beginTxn(ctx: IContext, opts?: TxnOptions): Promise<T>;
}
```
 
**Transactable: Go**
```go
// Respect existing "Tx" naming convention
type Transactable interface {
  BeginTx(ctx context.Context, opts *TxOptions) (Txn, error)
}
```

**Transactable: C#**
```cs
public interface Transactable<T> where T : Txn {
  Task<T> BeginTxn(IContext ctx, TxnOptions opts = null);
}
```
  
### `ChangeSet`

A ChangeSet is an in-memory transaction which simply accumulates a list of callbacks to invoke either on commit or rollback. 

```ts
interface ChangeSet extends Txn {
  defer(fn: (ctx: IContext) => Promise<void>): void;
  deferFail(fn: (ctx: IContext) => Promise<void>): void;
  commit(ctx: IContext): Promise<void>;
  rollback(ctx: IContext): Promise<void>;
}
```

All callbacks registered with `defer` are executed in order when `commit` is called. If any of them fail, or if `rollback` is called explicitly, then all the callbacks registered with `deferFail` are executed in order.

### `TxnChangeSet`

A TxnChangeSet combines both the in-memory ChangeSet and an underlying transaction, usually in a database.
 
```ts
 interface TxnChangeSet<T extends Txn> extends ChangeSet {
  deferTxn(fn: (ctx: IContext, txn: T) => Promise<void>): void;
  defer(fn: (ctx: IContext) => Promise<void>): void;
  deferFail(fn: (ctx: IContext) => Promise<void>): void;
  commit(ctx: IContext): Promise<void>;
  rollback(ctx: IContext): Promise<void>;
}
```

Callbacks registered with `deferTxn` will be run in a single underlying transaction. Callbacks registered with the base `defer` will be run only after the transaction, if needed, has successfully committed. Callbacks registered with `deferFail` are run if there are any errors in either the transaction or non-transaction callbacks, if committing the underlying transaction fails, or if `rollback` is called explicitly.

## Runners

With the generic [transaction interfaces](#basic-patternq) defined, it is possible to implement uniform transaction running algorithms.

A transaction running function accepts a context and an asynchronous callback. The callback itself is a function that accepts a context and a transaction of a given type. The implicit contract is:

- Before the callback is invoked, a new transaction is created and attached to the child context provided to the callback. If a transaction cannot be successfully created, the callback is never invoked
- If the callback is executed and completes successfully, then the transaction will be automatically committed
- If the callback is executed and fails, then the transaction will be automatically rolled back

### `TxnRunner`

The txn pattern exposes two complementary transaction running functions on a interface called TxnRunner:
 
**transaction runner: TypeScript**
```ts
interface TxnRunner<T extends Txn> {
  run(ctx: IContext, fn: (ctx: IContext, txn: T) => Promise<void>): Promise<void>;
  in (ctx: IContext, fn: (ctx: IContext, txn: T) => Promise<void>): Promise<void>;
}
```

**transaction runner: Go**
```go
type TxRunner[T Txn] interface {
  Run(ctx context.Context, fn: function(context.Context, T) error) error;
  In (ctx context.Context, fn: function(context.Context, T) error) error;
} 
```

**transaction runner: C#**
```cs
public interface TxnRunner<T> where T : Txn {
  Task Run(IContext ctx, Func<IContext, T, Task> fn)
  Task In (IContext ctx, Func<IContext, T, Task> fn)
}   
```

`run` and `in` differ in their behavior for nested transactions:

#### `run`

**run** requires that a new transaction be created and that the transaction be resolved on conclusion of the provided callback. 

If there is already a transaction on the provided context, and that existing transaction does not support nested transactions, then the call to run must fail.

#### `in`

**in** will reuse an existing transaction if one is present on the provided context, or will create a new one if there is not already a transaction on the provided context.

Note that if **in** reuses an existing transaction, then the nested call to **in** **WILL NOT** resolve the existing transaction, but will leave it open on conclusion of the callback. Whichever code created the existing transaction is responsible for committing it or rolling it back.

### Context

The contract of a [transaction runner](#runners) requires three context operations:

- **get** some [Transactable](#transactable-interface) instance from the context, which can be used to begin a new transaction
- **get** a [Txn](#txn-interface) instance from the context to detect nested transactions
- **set** a [Txn](#txn-interface) instance to make the transaction available to callbacks

The txn patterns describes the set of these three context getters and setters as a `TxnAccessor`:

**TxnAccessor: TypeScript**
```ts
interface TxnAccessor<T extends Txn> {
  getTransactable(ctx: IContext): Transactable<T> | null;
  getTxn(ctx: IContext): Txn | null;
  withTxn(ctx: IContext, txn: T): Context;
}
```

**TxnAccessor: Go**
```go
type TxnAccessor[T Txn] interface { 
  GetTransactable(ctx context.Context) Transactable[T]
  GetTxn(ctx context.Context) T
  WithTxn(ctx context.Context, txn T): context.Context
} 
```

**TxnAccessor: C#**
```cs
public interface TxnAccessor<T> where T : Txn {
  Transactable<T>? GetTransactable(IContext ctx);
  T? GetTxn(IContext ctx);
  IContext WithTxn(IContext ctx, T txn);
}
```
 
### Getting a TxnRunner

Implementations of the txn pattern provide factory methods for obtaining a [TxnRunner](#txnrunner) given an applicable [TxnAccessor](#context):

```ts
function txn(ctx: IContext): TxnRunner<Txn>;
function txn<T extends Txn>(acr: TxnAccessor<T>): TxnRunner<T>;
function changeSet(): TxnRunner<ChangeSet>;
function txnChangeSet(ctx: IContext): TxnRunner<TxnChangeSet<Txn>>;
function txnChangeSet<T extends Txn>(acr: TxnAccessor<T>): TxnRunner<TxnChangeSet<T>>;
```
  
The practical pattern for running a transaction then generally appears as follows:

```ts
txn(...).run((ctx, txn) => {
  // Logic to be run within the transaction
})

changeSet().in((ctx, cs) => {
  // Logic to be run within a possibly-reused ChangeSet
  cs.defer(...)
  cs.deferFail(...)
})
```

The overloads of `txn` and `txnChangeSet` that explicitly accept a `TxnAccessor` can also resolve the exact type of the `Txn`, but must also provide the context getters and setters every time a transaction runner is needed.

A more convenient approach is to use a context setter function that attaches the transaction accessors themselves to the context. The transaction accessors can then be retrieved internally by the implementations of `txn` and `txnChangeSet` that only require a context. The limitation is the the transaction runners returned by these factory methods cannot know the specific type of the `Txn`.

For a detailed example of both approaches, see [Example - MySQL]() in the documentation for the `@sabl/txn` package.

---
&copy; 2022 Joshua Honig. All rights reserved. See [LICENSE](../LICENSE.md).