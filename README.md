# sabl

[**sabl**](https://github.com/libsabl/patterns) is an open-source project to identify, describe, and implement effective software patterns which solve small problems clearly, can be composed to solve big problems, and which work consistently across many programming languages.

## Patterns

|Pattern|Purpose|
|-|-|
|[**context**](./patterns/context.md)|Inject state and dependencies and propagate cancellation signals|
|[**record**](./patterns/record.md)|Represent a data model at runtime, with no attached behaviors|
|[**txn**](./patterns/txn.md)|Implement transaction workflows agnostic of underlying storage type|
|[**storage pool**](./patterns/storage-pool.md)|Describe lifecycle expectations for pooled connections| 
|[**db api**](./patterns/db-api.md)|Provide a generic interface around relational databases.|

### Implementations

|Name|js/ts|golang|dotnet/c#|
|-|:-|:-:|:-:|
|[context](./patterns/context.md)|[@sabl/context](https://www.npmjs.com/package/@sabl/context)|[stdlib:context](https://pkg.go.dev/context)|-|
|[record](./patterns/record.md)|[@sabl/record](https://www.npmjs.com/package/@sabl/record)|-|-|
|[txn](./patterns/txn.md)|[@sabl/txn](https://www.npmjs.com/package/@sabl/txn)|-|-|
|[storage pool](./patterns/storage-pool.md)|[@sabl/storage-pool](https://www.npmjs.com/package/@sabl/storage-pool)|-|-|
|[db api](./patterns/db-api.md)|[@sabl/db-api](https://www.npmjs.com/package/@sabl/db-api)|-|-|

## License
Unless explicitly identified otherwise in the contents of a file, the content in this repository is licensed under the [Creative Commons Attribution-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-sa/4.0/legalcode) license. As summarized in the [Commons Deed](https://creativecommons.org/licenses/by-sa/4.0/) and strictly defined in the [license](./LICENSE.md) itself: you may share or adapt this work, provided you give appropriate credit, provide a link to the license, and indicate if changes were made. In addition, you must preserve the original license. For the details and actual legal license, consult the [license](./LICENSE.md).

### Source Code
The [license](#license) of this work is designed for cultural works, [not source code](https://creativecommons.org/faq/#can-i-apply-a-creative-commons-license-to-software). Any source code intended for more than an illustration should be distributed under an appropriate license by including that license in the applicable source code file, or by placing the source code file in another repository with an appropriate license.


---
&copy; 2022 Joshua Honig. All rights reserved. See [LICENSE](../LICENSE.md).