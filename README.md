# sabl

[**sabl**](https://github.com/libsabl) is an open-source project to identify, describe, and implement effective software patterns which solve small problems clearly, can be composed to solve big problems, and which work consistently across many programming languages.

## Patterns

|Name|golang|dotnet/c#|js/ts|
|-|:-:|:-:|:-:|
|[context](./patterns/context.md)|[stdlib](https://pkg.go.dev/context)|-|[**sabl**](./packages/js.md)|

## sabl philosophy

The patterns included in the sabl project reflect an overall approach to software engineering. At the root of that philosophy is an assertion:

<div style="font-size: 2rem; margin-top: 1em; padding: 1rem; margin-bottom: 1em; background: rgba(125,125,125,.1)">Engineers must understand the mechanics of the software they work on</div>

Engineering is a composition of and tension between **semantics** and **mechanics**, between **meaning** and **mathematics**, between **why** and **how**.  

Software programs are machines. Like the physical machines that humans have built for millennia, they are composed of meaningless physical operations which are arranged, connected, and ordered to achieve some meaningful purpose.

But where the demands and constraints of physical machines never allow engineers to escape the immediacy of the underlying mechanics, software systems have allowed abstraction and dissociation of a totally different nature. And with this has come a fantasy that mechanics can be ignored or delegated in favor of semantics. It promises productivity and even happiness, but cannot escape the following basic logic:

- Software is written and managed by humans.
- In order to effectively build, manage, maintain, modify, or otherwise work with software (or any other machine), a human working on that software must be able to **accurately and completely imagine how the software will operate**.
- Operation is the outcome of mechanics. In other words, the human must completely understand the mechanics of the software.

Conversely

- If a human does not completely understand the mechanics of a machine, including software
- Then it is not possible for them to imagine the full range of behaviors or states that the machine will exhibit
- And it is therefore impossible to know whether the machine will fulfill its intended **purpose** by exhibiting completely and solely **intended behaviors**

Humans make mistakes. The more imagination required, the more mechanical mistakes and misunderstandings that occur. Patterns that require a great deal of imagination about underlying mechanics necessarily lead to greater degrees of failure to produce desired behaviors, and greater degrees of "success" in producing undesired or unexpected behaviors.

### All the things

The problem, at its surface, is that there are too many things to know. It can seem like programmers and engineers must know a vast range of things in order to start working at all. There are at least two ways to address this. One fails, the other succeeds.

The widespread non-solution is to try to eliminate the need to understand mechanics by replacing them with more human-friendly semantics, to allow program authors to simply say what they want, and not worry about how it is achieved. In practice these approaches not only obfuscate the simple mechanics they are meant to facilitate, but also cause mechanics to become exponentially more complicated.

The worst of these patterns allow program authors to cause a huge range of state or behavior changes with very few keystrokes, or to completely separate the program-words which request a behavior from the places where the behaviors are implemented. This can be exciting and powerful, especially for newer programmers. Reward centers in our human brains pump out reinforcing dopamine for discovering an amazing exchange of small input for huge output.

But it is the opposite of what engineers actually need to achieve: the **minimum possible** range of states and behaviors that fulfill the semantic objective, and the ability to accurately imagine that range of states. Those "developer happiness" instincts evolved for gathering calories, not for building reliable mechanical systems. Kill one mammoth, tribe eats for a month. Add one `before-calculate-property-changes-but-after-walk-dependencies-graph` hook, bug hunt for a year. But only when inexplicable behaviors in an apparently unrelated feature start to appear at seemingly random times, but especially on weekends and holidays.

Engineers still get to enjoy that reward, but we must train our brains to perceive it in the exchange of limited but concerted efforts to build a highly reliable == highly understandable system in order to enjoy its durable and predictable operation over indefinitely large scales in volume, time, and intentional behavior changes (e.g. new features).

### Things that work

The good news is that, after some 70 years of software programming, humans have collectively invented or discovered many patterns that allow the design and construction of software systems that succeed in maintaining mechanical clarity while also reducing the range of concepts that must be understood within a particular domain. 

These patterns reflect fundamental physics of how information can be constrained or partitioned. They have analogs and equivalents in control theory, machine building, physics, and even biology.
 
The actual solution to the problem of too many things to know is to eliminate the need to understand some mechanics by making them impossible. At best, by explicitly describing only the mechanics that are allowed.

A good example of this is the `go` standard library's core `io` interfaces.

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}
```

These simple interfaces for describing how sequences of bytes may be read or written eliminate the need to understand any other mechanics of how some hardware or software layer my produce or consume those bytes by making all those mechanics **irrelevant**. 

This defines a clear partition in concept space between the how and the what. Implementers of a `Reader` or `Writer` must certainly understand their own internal mechanics, but all those internal mechanics must be resolved to a tiny range of observable behaviors. Consumers of these interfaces don't need and in fact are not able to know about the internal implementation details.

This particular example does include one opportunity to snatch defeat from the jaws of victory: that little `err` return value. Authors of go programs can and sometimes do use that to examine and depend on internal details of a particular io implementation. Whether or not such internals are irrelevant, or are in fact the legitimate subject matter of the code in question, sits squarely in the human-semantic realm of learned pattern recognition. That is the truly human and creative work of engineering.

## License
Unless explicitly identified otherwise in the contents of a file, the content in this repository is licensed under the [Creative Commons Attribution-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-sa/4.0/legalcode) license. As summarized in the [Commons Deed](https://creativecommons.org/licenses/by-sa/4.0/) and strictly defined in the [license](./LICENSE.md) itself: you may share or adapt this work, provided you give appropriate credit, provide a link to the license, and indicate if changes were made. In addition, you must preserve the original license. For the details and actual legal license, consult the [license](./LICENSE.md).

### Source Code
The [license](#license) of this work is designed for cultural works, [not source code](https://creativecommons.org/faq/#can-i-apply-a-creative-commons-license-to-software). Any source code intended for more than an illustration should be distributed under an appropriate license by including that license in the applicable source code file, or by placing the source code file in another repository with an appropriate license.


---
&copy; 2022 Joshua Honig. All rights reserved. See [LICENSE](../LICENSE.md).