sabl / [patterns](../README.md#patterns) / record

# record
 
**record** is a pattern for representing a data model at runtime. It uses record or model instances exclusively to hold the attributes of a single flat row of data and to allow on-record caching of related records. Even with cached relations, records are passive and do not hold a connection to or knowledge of where the record was loaded from.

Conversely, records do provide features that communicate which attributes may be updated and when -- only when loaded, only indirectly through a mutator method, or any time simply by assigning to a property. These features express the intent of the underlying data model and aid authors in writing correct programs using that data model.

### Implementations
  
Most aspects of the record pattern are indeed *patterns* that authors implement in their own code bases. sabl does provide implementations of common building blocks such as cached relations.

|Language|Category|Source|Package|
|-|-|-|-|
|JS / TS|sabl|github : [libsabl/record-js](https://github.com/libsabl/record-js/blob/main/src)|[@sabl/record](https://www.npmjs.com/package/@sabl/record)|
 
## Basic pattern

### Features

The record pattern consists of three complementary features:

1. [Attribute modes](#attributes-and-mutability)
2. [Relations](#cached-relations)
3. [Initializers](#initializers)

[Attribute modes](#attributes-and-mutability) communicate and enforce the circumstances in which attribute values may be **set or updated**. [Relations](#cached-relations) provide consistent mechanics for caching and retrieving related records. [Initializers](#initializers) provide a consistent patterns for updating the internal state of a record when loading attribute values or related objects from an external data source.

### Attributes and mutability

A key feature of the record pattern is to explicitly communicate and enforce when particular attributes may be set or updated. All attributes can always be read by a client; the only difference is in when the attributes can be written. The techniques used to allow or prevent write access to certain attributes varies by the target language. 

The following are the four attribute modes, from least to most restrictive:

|Mode|When writable|
|-|-|
|simple / public|By client at any time, usually through direct assignment|
|protected-set|By client using a mutator method|
|write-once|By client but only when creating a record|
|generated / read-only |Only by remote storage|

#### simple / public

Attributes that may be updated at any time with no particularly concerning consequences can simply be public fields modified by direct assignment.
 
#### protected-set and mutators

These attributes may be updated by a client, but for semantic or human reasons the author of the data model wants to cause authors of code which use a record to stop and think before modifying the values of the applicable property. This is the programmatic equivalent of a flip-up toggle switch cover. In this case updating the attribute value is allowed, but an additional manual step is require to prevent accidental or careless changes. Thus the properties are readonly to prevent simple assignment, but **mutator** methods are provided in order to update the values.

Mutator methods may contain simple checks on the basic validity of data, but should never include business rules. Appropriate validity checks could include ensuring a string is non-empty or not too long for the intended data store field, or ensuring an integer is a known enumeration value.
 
Mutator methods may also be used to maintain correctness between correlated attributes. For example, invoking `setQuantity` or `setPrice` on some invoice line record might also update the value of the computed `extendedAmount` field.

Mutator methods cannot perform any service-dependent validation, such as verifying that a value does not violate a unique constraint. They should also never modify data on any other record, even if those records are accessible through a cached relation. In other words, validation of interaction between a record and *other records* either in the runtime or underlying storage is explicitly and by design not what mutators are for in this pattern.

#### write-once
  
These attributes are supplied by a client when first creating a record, but they may never be changed afterwards. This is a logical concept that does not exist in any database system but is often important for the correct use of business data. The values can only be mechanically set on a record instance using its initializer.

#### generated / read-only / computed

These attributes are derived or set by the external data source. The values can only be mechanically set on a record instance using its initializer.

### Cached relations

It is common in data-intensive applications to need to easily navigate from one record to related records, and just as importantly to be able cache references to related records. Being able to explicitly assign a related record is also necessary. All together, these features allow efficient de-normalization, serialization, and data navigation without incurring prohibitive n+1 query costs. Note that the record pattern itself has no opinion about how records are retrieved from a data source, it simply provides a consistent pattern to cache related records.

#### Async retrieval

The patterns described here work whether or not the mechanisms used to retrieve related records are asynchronous, but it is strongly recommended that implementations use asynchronous methods for retrieving related records. The [implementations](#implementations) in the sabl libraries always use asynchronous methods for record retrieval. This precludes exposing related records as *properties*, which are intrinsically and semantically synchronous. 

Modern programming languages that support asynchronous patterns also include internal optimizations to ensure that asynchronous methods which complete synchronously continue within the same frame or thread and incur minimal overhead. In this case, calling an asynchronous relation getter method will complete synchronously if the related record has already been cached.

The initializer pattern also provides simple APIs that allow synchronous checks for whether related records are already cached and, if so, to synchronously retrieve the cached values.

Put another way: 
- In order to fetch a related record, client code must always `await` a method, even if the related record or collection is already cached. 
- In some languages, notably `go`, all methods are implicitly asynchronous and all calls are implicitly `awaited`. Even in languages where an explicit `await` is required, it is still fairly cheap to await a result that completes synchronously. 
- Finally, hyper-optimization is possible (though generally not necessary) by first synchronously checking the internal state of the relation via the record's initializer.

#### Context injection

An important goal of the record pattern is to ensure that record instances are both passive and origin-agnostic. A record should neither know or retain knowledge about where attribute values or related records were retrieved from. 

And yet, retrieving a related record requires interacting with some remote or in-memory data source or service. This apparent contradiction is resolved using the [context pattern](./context.md). Relation getter methods should always accept a context as the first argument. Any necessary services needed to fetch the related record or set of records can be retrieved from this injected context. For an illustration, see [relation implementation](#relation-implementation) below.

### Initializers

Three of the four [attribute modes](#attributes-and-mutability) require that attribute properties prevent direct assignment of values. And yet the values must be set somehow when loading data onto a recod instance. The initializer pattern provides a consistent pattern for mechanically setting these attribute values in the context of loading data from an external data source.

All initialization and cache inspection APIs are consolidated under a single method or read-only property `init` which returns an object with a `load` method which accepts the scalar values of all attributes on the record.

#### Initial loading

The `load` method should include logic to check whether the target record has already been initialized. If it has, the `load` method should throw an exception or return an error. This prevents simple mistakes where client code attempts to load more than one record of data to the same in-memory object. A simple implementation is to simply check if the primary key attribute or tuple still has its default / zero value.

#### Reloading

Optionally, authors may support refreshing an existing record instance with updated data by providing an overload or parameter of the `load` method that allows a client to explicitly indicate that they are reloading an existing record. This is useful for refreshing in-memory attribute values from a remote data store while also maintaining an in-memory object graph of related records. 

If authors choose to support, this, they should if possible validate that the updated data is for the correct record by comparing the primary key attribute value or tuple to the values already present on the record instance.

#### Relations

If a record has relations, the relation objects themselves should or can be exposed on the initializer object. This allows client code to explicitly check, set, or invalidate cache state. This is essential for enabling efficient eager loading of related objects to avoid unnecessary n+1 type queries to the origin data store.

## Example

The following example shows a complete TypeScript example of related Invoice and InvoiceLine records. Though contrived, they illustrate essentially all features of the record pattern. Note that the record pattern is not specific to TypeScript / JavaScript, and future examples will show implementations in other languages.

### Definition

All features of a record can be described through interfaces. Authors may or may not chose to explicitly define interfaces and then separate implementations for each record type. Likewise authors may or may not choose to split out record interfaces into individual components as illustrated below. In this example explicit and incremental interfaces are used to most clearly illustrate and distinguish the features of a record type.

**invoice.ts**
```ts
import type { CollectionRelation, RecordOf } from '@sabl/record';
import type { IContext } from '@sabl/context';
import type { InvoiceLine } from './invoice-line';

/** The properties of an Invoice record */
export interface InvoiceProps {
  readonly id: number;             // key, generated
  readonly invoiceNumber: number;  // protected-set
  label: string;
}

/** The mutators for protected properties of an Invoice record */
export interface InvoiceMutators {
  /** Change the invoice number */
  setInvoiceNumber(value: number): void;
}

/** The relations of an Invoice record */
export interface InvoiceRels {
  getLines(ctx: IContext): Promise<InvoiceLine[]>;
}

/** Access to set internal state of the record.
 * Includes the required load method as well as 
 * explicit access to the lines relation. */
export interface InvoiceInitter {
  load(data: InvoiceProps, refresh: boolean): void;
  readonly lines: CollectionRelation<Invoice, InvoiceLine>;
}

/** The Invoice is a composition of the props, the mutators,
 * the relations, and a read-only `init` property which
 * returns the given record's initializer. It also
 * implements the simple RecordOf interface. */
export interface Invoice extends 
  InvoiceProps, 
  InvoiceMutators, 
  InvoiceRels, 
  RecordOf<number> {
  readonly init: InvoiceInitter;
}
```

**invoice-line.ts**
```ts
import type { RecordOf, Relation } from '@sabl/record';
import type { IContext } from '@sabl/context';
import type { Invoice } from './invoice';

/** The properties of the InvoiceLine record */
export interface InvoiceLineProps {
  readonly id: number;           // key, generated
  readonly invoiceId: number;    // write-once
  readonly product: string;      // protected-set
  readonly quantity: number;     // protected-set
  readonly price: number;        // protected-set
  readonly amount: number;       // calculated
}

/** The mutators for protected properties of the InvoiceLine record */
export interface InvoiceLineMutators {
  /** Change the product. */
  setProduct(value: string): void;

  /** Change the quantity number. Also updates amount */
  setQuantity(value: number): void;

  /** Change the price. Also updates amount */
  setPrice(value: number): void;
}

/** The relations of the InvoiceLine record */
export interface InvoiceLineRels {
  getInvoice(ctx: IContext): Promise<Invoice>;
}

/** Initter for InvoiceLine record */
export interface InvoiceLineInitter {
  load(data: InvoiceLineProps, refresh: boolean): void;
  readonly invoice: Relation<number, Invoice>;
}

/** An InvoiceLine record */
export interface InvoiceLine extends 
  InvoiceLineProps,
  InvoiceLineMutators,
  InvoiceLineRels,
  RecordOf<number> {
  readonly init: InvoiceLineInitter;
}
```

### Implementation

#### Relation

*Implementers* of the record must also provide implementations of related record retrieval operations, though these implementations themselves should generally just retrieve some repository or data source *interface* from the [context](./context.md). The repository interface will describe how to ask for a particular record or set of records, but does not require the consumer to know about the implementation of retrieval. At the very least this allows for simple mock implementations to facilitate testing without the need for an actual data store.

##### Repository

Here we use an extremely simple read-only repository interface for example purposes only. It does reflect the following conventions which hold true even for complete real programs: 

- Record operations should accept a [context](./context.md) so that dependencies can be injected
- Record operations should be asynchronous
- Retrieval operations for a single record can return an empty result
- If a record operation could return more than one record then such methods should return asynchronous iterators so that each record can be incrementally awaited, or the entire operation aborted, without having to wait for an entire result set.

Following the [context pattern](./context.md), we also provide a context getter and setter for the repository interface:

**repository.ts**
```ts
import { IContext, Context, withValue } from '@sabl/context';
import { Maybe } from '@sabl/context';

export interface Repository {
    getInvoice(ctx: IContext, id: number): Promise<Invoice | null>;
    getInvoiceLines(ctx: IContext, invoice: Invoice): AsyncIterable<InvoiceLine>;
} 

const ctxKeyRepo = Symbol('repo');

export function withRepo(ctx: IContext, repo: Repository): Context {
  return withValue(ctx, ctxKeyRepo, repo);
}

export function getRepo(ctx: IContext): Maybe<Repository> {
  return <Maybe<Repository>>ctx.value(ctxKeyRepo);
}
```

##### Stream, Collect

Authors have two options for collection relations. The retrieval method could return an async iterator, just as underlying repository methods should, or the retrieval method could return a complete collection. The sabl library [implementations](#implementations) make the latter choice. This does require converting between an async iterator and a simple asynchronous method which returns a complete collection. Alternatively, if an author chooses to return collection relations as async iterators, then to return related records that are already cached they will have to convert from a simple collection to an async iterator. Where applicable, sabl record libraries provide `stream` and `collect` methods which accomplish this:

- `stream` iteratively yields the individual members of a collection as an async iterator
- `collect` awaits all items from a source async iterable and then resolves the completed collection

##### Relation implementation

Finally, here are the example implementations of the single-record invoice relation and the collection relation from an invoice to its child invoice lines. For clarity, the methods for retrieving the invoice or invoice lines are defined separately and then referenced from the applicable relation constructors.

**relations.ts**
```ts
import { collect, CollectionRelation, NullableRelation, Relation } from '@sabl/record';
import { Context, IContext } from '@sabl/context';

function getInvoice(ctx: IContext, key: number): Promise<Invoice | null> {
  const repo = Context.as(ctx).require(getRepo);
  return repo.getInvoice(ctx, key);
}

function getInvoiceLines(
  ctx: IContext,
  parent: Invoice
): Promise<Iterable<InvoiceLine>> {
  const repo = Context.as(ctx).require(getRepo);
  return collect(repo.getInvoiceLines(ctx, parent));
}

/** A relation to an invoice record */
export class InvoiceRelation extends Relation<number, Invoice> {
  constructor(keyProp: string) {
    super(keyProp, getInvoice);
  }
}

/** A nullable relation to an invoice record */
export class InvoiceRelation extends NullableRelation<number, Invoice> {
  constructor(keyProp: string) {
    super(keyProp, getInvoice);
  }
}

/** A relation of an invoice to its lines */
export class InvoiceLinesRelation extends CollectionRelation<
  Invoice,
  InvoiceLine
> {
  constructor() {
    super(getInvoiceLines);
  }
}
```

#### Record

With the relations defined relative to a known repository interface, we can now define the record implementations themselves. Once again, the examples here separately define the [record interfaces](#definition) and the concrete record implementations, but authors could choose to skip the interface definitions. Records are by design source agnostic, so even for test data there is no need to create mock record classes. Instead, actual record instances are initialized and then simply loaded with mock data.

**invoice-record.ts**
```ts
import { IContext } from '@sabl/context';
import { RecordError } from '@sabl/record';
import { Invoice, InvoiceProps } from './invoice';
import { InvoiceLine } from './invoice-line';
import { InvoiceLinesRelation } from './relations';

/** An invoice */
export class InvoiceRecord implements Invoice {
  static readonly typeName = 'example:invoice';
   
  // ====================
  //  Attributes
  // ====================
  #id: number = null!;
  /** The primary key id of the record `[key,generated]` */
  get id(): number {
    return this.#id;
  }

  #invoiceNumber: number = null!;
  /** The document number `[write-once]` */
  get invoiceNumber(): number {
    return this.#invoiceNumber;
  }
 
  // ====================
  //  Relations
  // ====================
  readonly #lines = new InvoiceLinesRelation();

  getLines(ctx: IContext): Promise<InvoiceLine[]> {
    return this.#lines.getItems(ctx, this);
  }
    
  // ====================
  //  Initter
  // ====================
  static readonly Initter = class Initter {
    readonly #record: Invoice;

    constructor(record: Invoice) {
      this.#record = record;
    }

    get lines() {
      return this.#record.#lines;
    }

    load(data: InvoiceProps, refresh = false) {
      const r = this.#record;
      if (refresh) {
        if (data.id !== r.#id) {
          throw new RecordError(RecordError.WRONG_RECORD);
        }
      } else {
        if (r.#id != null!) {
          throw new RecordError(RecordError.REINITIALIZED);
        }
        r.#id = data.id;
      }
      r.#invoiceNumber = data.invoiceNumber;
    }
  };

  readonly #initter = new InvoiceRecord.Initter(this);

  get init() {
    return this.#initter;
  }

  // ====================
  //  RecordOf
  // ====================
  getKey(): number {
    return this.#id;
  }

  getType(): string {
    return InvoiceRecord.typeName;
  }
}
```

**invoice-line-record.ts**
```ts
import { IContext } from '@sabl/context';
import { RecordError } from '@sabl/record';
import { Invoice } from './invoice';
import { InvoiceLine, InvoiceLineProps } from './invoice-line';
import { InvoiceRelation } from './relations';

/** An invoice line */
export class InvoiceLineRecord implements InvoiceLine {
  static readonly typeName = 'example:invoice-line';

  // ====================
  //  Attributes
  // ====================
  #id: number = null!;
  /** The primary key id of the record `scio:[key,generated]` */
  get id(): number {
    return this.#id;
  }

  #product: string = null!;
  /** The product name `[protected-set]` */
  get product(): string {
    return this.#product;
  }

  #quantity: number = null!;
  /** The line item quantity `[protected-set]` */
  get quantity(): number {
    return this.#quantity;
  }

  #price: number = null!;
  /** The price per unit `[protected-set]` */
  get price(): number {
    return this.#price;
  }

  #amount: number = null!;
  /** The line item amount `[calculated]` */
  get amount(): number {
    return this.#amount;
  }
  
  // ====================
  //  Mutators
  // ====================

  /** Change the product. */
  setProduct(value: string): void {
    this.#product = value;
  }

  /** Change the quantity. Also updates amount */
  setQuantity(value: number): void {
    this.#quantity = value;
    this.#amount = this.#quantity * this.#price;
  }

  /** Change the price. Also updates amount */
  setPrice(value: number): void {
    this.#price = value;
    this.#amount = this.#quantity * this.#price;
  }

  // ====================
  //  Relations
  // ====================
  readonly #invoice = new Invoice.Relation('invoiceId');

  /** The id of the {@link Invoice} for this record */
  get invoiceId() {
    return this.#invoice.key;
  }

  /** Lazy-loaded reference to the {@link Invoice} for this record */
  getInvoice(ctx: IContext) {
    return this.#invoice.getItem(ctx);
  }

  // ====================
  //  Initter
  // ====================
  static readonly Initter = class Initter {
    readonly #record: InvoiceLine;

    constructor(record: InvoiceLine) {
      this.#record = record;
    }

    get invoice() {
      return this.#record.#invoice;
    }

    load(data: InvoiceLineProps, refresh = false) {
      const r = this.#record;
      if (refresh) {
        if (data.id !== r.#id) {
          throw new RecordError(RecordError.WRONG_RECORD);
        }
      } else {
        if (r.#id != null!) {
          throw new RecordError(RecordError.REINITIALIZED);
        }
        r.#id = data.id;
        r.#invoice.initKey(data.invoiceId);
      }
      r.#product = data.product;
      r.#quantity = data.quantity;
      r.#price = data.price;
      r.#amount = data.amount;
    }
  };

  readonly #initter = new InvoiceLineRecord.Initter(this);

  get init() {
    return this.#initter;
  }

  // ====================
  //  RecordOf
  // ====================
  getKey(): number {
    return this.#id;
  }

  getType(): string {
    return InvoiceLineRecord.typeName;
  }
}
```

---
&copy; 2022 Joshua Honig. All rights reserved. See [LICENSE](../LICENSE.md).