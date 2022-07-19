sabl / [patterns](../README.md#patterns) / db api

# db api
 
<small>depends on: [**context**](./context.md)</small><br>
<small>references: [**txn**](./txn.md), [**storage-pool**](./storage-pool.md)</small>
 
**db api** is a simple, [context](./context.md)-aware pattern for describing interactions with a relational database. It is compatible with the more general connection pooling and transaction patterns described in [**storage-pool**](./storage-pool.md) and [**txn**](./txn.md).

The patterns described here are based on the [golang](https://go.dev/doc/) standard library [`database/sql` package](https://pkg.go.dev/database/sql).

## Basic Pattern

The original golang [`database/sql` package](https://pkg.go.dev/database/sql) summarizes all interactions with relational databases to a concise set of interfaces that work the same regardless of the underlying database platform. That package combines connection pooling, transaction workflows, and basic SQL CRUD APIs in a set of types such as `DB`, `Conn`, `Stmt`, and `Tx`. 

The sabl project describes connection pooling in general in [**storage-pool**](./storage-pool.md), and transaction workflows in general in [**txn**](./txn.md). What remains here is the common API for SQL-mediated interactions with a relational database.

This entire API boils down to three methods, with three corresponding return types:

|Method|Returns|SQL Verb|
|-|-|-|
|`query`|`Rows`|`SELECT`|
|`queryRow`|`Row`|`SELECT`|
|`exec`|`Result`|`INSERT`, `UPDATE`, `DELETE`, `EXEC`, etc| 

All three methods accept a [**context**](./context.md), a string SQL statement, and a variable array of SQL parameter values:

**TypeScript**
```ts
interface DbApi {
  query   (ctx: IContext, sql: string, ...params: any[]): Promise<Rows>
  queryRow(ctx: IContext, sql: string, ...params: any[]): Promise<Row>
  exec    (ctx: IContext, sql: string, ...params: any[]): Promise<Result> 
}
```

**Go**
```go
type DBAPI {
    Query   (ctx context.Context, sql string, args ...any) (*sql.Rows , error)
    QueryRow(ctx context.Context, sql string, args ...any) (*sql.Row  , error)
    Exec    (ctx context.Context, sql string, args ...any) (sql.Result, error)
}
```

**C#**
```cs
public interface DbApi {
  Task<IRows>   Query   (IContext ctx, string sql, params object[] params)
  Task<IRow >   QueryRow(IContext ctx, string sql, params object[] params)
  Task<IResult> Exec    (IContext ctx, string sql, params object[] params)
}
```

## Result Types

### Result

A `Result` is just a wrapped for a count of affected rows and, where supported, the most recent inserted row id.

**TypeScript**
```ts
interface Result {
  readonly rowsAffected: number;
  readonly lastId: number | undefined;
}
```

**Go**: Defined in [`database/sql`](https://pkg.go.dev/database/sql): see [**Result**](https://pkg.go.dev/database/sql#Result)

```go 
type Result interface { 
	LastInsertId() (int64, error) 
	RowsAffected() (int64, error)
}
```

**C#**
```cs
public interface Result {
  long RowsAffected { get; }
  long LastInsertId { get; }
}
```

### Row

A `Row` represents a single row of data already received. The original go package defines it as a concrete `struct`. sabl packages use an interface instead:
 
**TypeScript**
```ts
interface Row { 
  [key: string | symbol]: any; 
  [index: number]: any;
}
```

**Go**: Defined in [`database/sql`](https://pkg.go.dev/database/sql): see [**Row**](https://pkg.go.dev/database/sql#Row)

```go 
type Row struct { 
    Err() error
    Scan(dest ...any) error
}
```

**C#**
```cs
public interface Row {
  object this[string key] { get; }
  object this[int index] { get; }
}
```

### Rows
 
A `Rows` represents a database cursor which can be advanced through a result set. It is the responsibility of clients to ensure `Rows` objects are closed. The original go package defines it as a concrete `struct`. sabl packages use an interface instead. The go package provides access to the current row values with the `Scan` method, while sabl packages provide access to the current row as a `Row` interface which can retrieve individual column values by either name or ordinal.
 
**TypeScript**
```ts
interface Rows extends AsyncIterable<Row> { 
  close(): Promise<void>; 
  next(): Promise<boolean>; 
  get columns(): string[];
  get columnTypes(): ColumnInfo[]; 
  get row(): Row;
  get err(): Error | null;
}
```

**Go**: Defined in [`database/sql`](https://pkg.go.dev/database/sql): see [**Rows**](https://pkg.go.dev/database/sql#Rows)

```go 
type Rows struct {
    Close() error
    ColumnTypes() ([]*ColumnType, error)
    Columns() ([]string, error)
    Err() error
    Next() bool
    NextResultSet() bool
    Scan(dest ...any) error
}
```

**C#**
```cs
public interface Rows : IAsyncEnumerable<Row> {
  Task          Close();
  Task<bool>    Next();
  string[]      Columns { get; }
  ColumnInfo[]  ColumnTypes { get; }
  Row?          Row { get; }
  Exception?    Err { get; }
}
```