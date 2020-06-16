# Goals

Living Document. Michal Srb, 2020-06.

There are sixteen ordered goals that the smart step syntax tries to fulfill,
which may be summarized,\
“Don’t break my code,”\
“Don’t make me overthink,”\
“Don’t shoot me in the foot,”\
“Make my code easier to read,”\
and a few other goals.

<table>
<tr>
<td>

**“Don’t break my code.”**

1.  [Backward compatibility](#backward-compatibility)
2.  [Zero runtime cost](#zero-runtime-cost)
3.  [Forward compatibility](#forward-compatibility)

<td>

**“Don’t make me overthink.”**

4.  [Syntactic locality](#syntactic-locality)
5.  [Semantic clarity](#semantic-clarity)
6.  [Expressive versatility](#expressive-versatility)
7.  [Cyclomatic simplicity](#cyclomatic-simplicity)

<td>

**“Don’t shoot me in the foot.”**

8.  [Simple scoping](#simple-scoping)
9.  [Static analyzability](#static-analyzability)

<tr>
<td>

**“Make my code easier to read.”**

11. [Untangled flow](#untangled-flow)
12. [Distinguishability](#distinguishability)
13. [Terse parentheses](#terse-parentheses)
14. [Terse variables](#terse-variables)

<td>

**Other**

15. [Human writability](#human-writability)
16. [Novice learnability](#novice-learnability)

</table>

## “Don’t break my code.”

The syntax should not break any existing code; it should also be forward
compatible with future code.

### Backward compatibility

The syntax must avoid stepping on the toes of existing code, including but not
limited to JavaScript libraries such as [jQuery][] and [Lodash][]. In particular,
the topic reference should not be an existing identifier such as `$` or `_`,
which both may cause surprising results to a developer who adopts pipelines
while also using a globally bound convenience variable. It is a common pattern
to do this even without a library: `var $ = document.querySelectorAll`”. The
silent shadowing of such outer-context variables may silently cause bugs, which
may also be difficult to debug (see [Expressive Versatility][]).

Nor can it cause previously valid code to become invalid. This includes, to a
lesser extent, common nonstandard extensions to JavaScript: for instance, using
`<>` for the topic reference would retroactively invalidate existing E4X and JSX
code.

This proposal uses `#` for its topic reference. This is compatible with all
known previous JavaScript code. `?` and `@` could be chosen instead, which are
each also backwards compatible.

### Zero runtime cost

This could be considered a specific type of backward compatibility. When
translating old code into the new syntax, doing so should not cause unexpected
performance regression. For instance, the new syntax should not require memory
allocation for newly created functions that were not necessary in the old code.
Instead, it should, at least theoretically, perform as well the old code did for
both memory and CPU. And it should be able to do this without dramatically
rearranging code logic or relying on hidden, uncontrollable compiler
optimization.

For instance, in order to apply the syntax to the logic of an async functions, a
hypothetical new pipeline syntax might not support using `await` in the same
async context as the pipeline itself. Such a syntax would, for each of its
pipelines’ steps, require inner async functions that would return wrapper
promises and pass them between consecutive steps. Such an approach would be be
unnecessarily expensive to naively evaluate for both CPU and memory. But
inlining these async functions may be internally complicated, and such
optimizations would be difficult for the developer to correctly predict and
might differ widely between JavaScript engines.

Instead, this proposal’s use of a topic reference enables the zero-cost
rewriting of any expression within the current environmental context, including
`await` operations in async functions, without having to create unnecessary
inner async functions, and without having to wrap values in unnecessary promises.

### Forward compatibility

The syntax should not preclude other proposals: both [already-proposed
ECMAScript proposals][other ecmascript proposals], such as [partial function
application][] and [private class fields][] – as well as the [Additional
Features][intro] of this proposal. The Core Proposal is forward compatible with
all of these, especially because of its [early errors][].

## “Don’t shoot me in the foot.”

The syntax should not be a footgun: it should not easy for a developer to
accidentally shoot themselves in the foot with it. Lessons from the [topic
variables of other programming languages][topic references in other programming languages] may be instructive in formulating these goals.

### Opt-in behavior

The lexical topic is implicit, hidden state. State is intrinsically dangerous in
that it may induce the developer to commit [mode errors][], later “surprising”
the developer with unpredicted behavior. It should therefore not be easy for a
developer to accidentally bind or use the topic. If the developer accidentally
binds or uses the topic, any use of that reference could result in subtle,
pernicious bugs.

The larger the probability that a developer will accidentally clobber or
overwrite the topic, the less predictable their code becomes. It should not be
easy to accidentally shadow a reference from an outer lexical scope.

The larger the probability that a developer accidentally uses the topic, the
less predictable their code becomes. It should not be easy to accidentally use
the current lexical scope’s topic. In particular, bare/tacit function calls that
use the topic should not be easy to accidentally perform.

In this proposal, the developer therefore **must explicitly opt into**
topic-using behavior, whether binding or using, by using the pipe operator
`|>`. This includes [Additional Feature TS][], which requires the use of `|>`.

This is quite different than [much prior art in other programming
languages][topic references in other programming languages]. Other languages
frequently bind their topic references using numerous syntactic structures, with
no way for the developer to opt out. In addition, bare/tacit function calls are
easier to accidentally perform in some programming languages – making it more
difficult to tell whether a bare identifier `print` is meant to be a simple
variable reference or a bare function call on the topic value.

### Simple scoping

It should not be easy to accidentally shadow a reference from an outer lexical
scope. When the developer does so, any use of that reference could result in
subtle, pernicious bugs.

The rules for where the topic is bound should be simple and consistent. It should
be clear and obvious when a topic is bound and in what scope it exists. And
forgetting these rules should result in [early, compile-time errors][early errors], not subtle runtime bugs.

It should always be easy to find the origin of a topic binding, without looking
deeply into the stack. Topic references are therefore bound only in the steps
of pipelines, and they cannot be used within `function`, `class`, `for`,
`while`, `catch`, and `with` statements (see [Core Proposal][]). When the
developer wishes to trace the origin of a topic binding, they may be certain
that if they find any of such statements during their search, they have moved
too far and should retrace their path for the topic binding.

This proposal’s topic references are different than [much prior art in other
programming languages][topic references in other programming languages]. Other
languages frequently use dynamic binding rather than lexical binding for their
topic references.

### Static analyzability

[Early errors][] help the editing JavaScript developer avoid common [footguns][]
at compile time, such as preventing them from accidentally omitting a topic
reference where they meant to put one. For instance, if `x |> 3` were not an
error, then it would be a useless operation and almost certainly not what the
developer intended. Situations like these should be statically detectable and
cause compile-time [early errors][].

The same preference for strict early errors is used by the class decorators
proposal: see [tc39/proposal-decorators#30][], [tc39/proposal-decorators#42][],
and [tc39/proposal-decorators#60][]. Early errors also assist with [forward
compatibility][], as changing a behavior from “throws” to “does something” is
generally web compatible, though the reverse is not true.

## “Don’t make me overthink.”

The syntax should not make a developer overthink about the syntax, rather than
their product.

### Syntactic locality

The syntax should minimize the parsing lookahead that the compiler must check.
If the grammar makes [garden-path syntax][] common, then this increases the
dependency that pieces of code have on other code. This long lookahead in turn
makes it more likely that the code will exhibit developer-unintended behavior.

By having a single style that requires the topic reference syntax becomes more readable and predictable.

### Semantic clarity

<table>
<tbody>
<tr>
<td colspan=2>

The syntax might be unambiguous, but its semantics can be more difficult to interpret.

<tr>
<td>

For instance:

```js
input |> object.method;
```

…may be clear enough.

<td>

It means:

```js
object.method(input);
```

<tr>
<td>

But:

```js
input |> object.method(); 🚫
```

…is less clear.

<td>

It could reasonably mean either of these lines:

```js
object.method(input);
object.method()(input);
```

<tr>
<td>

Adding other arguments:

```js
input |> object.method(x, y); 🚫
```

…makes it worse.

<td>

It could reasonably mean any of these lines:

```js
object.method(input, x, y);
object.method(x, y, input);
object.method(x, y)(input);
```

<tr>
<td>

And this is even worse:

```js
input |> await object.method(x, y); 🚫
```

<td>

It could reasonably mean any of these lines:

```js
await object.method(input, x, y);
await object.method(x, y, input);
await object.method(x, y)(input);
(await object.method(x, y))(input);
```

<tr>
<td colspan=2>

It is undesirable for the human reader to be uncertain which of multiple
interpretations of a pipeline – all which are reasonable – is correct. It is
both a distracting incidental cognitive burden and a potential source of
developer error. [The Zen of Python][pep 20] famously says, “Explicit is better
than implicit,” for reasons such as these. And it is for these reasons that this
proposal makes the unclear pipelines above [early errors][].

<tr>
<td>

This pipeline:

```js
input |> object.method(#);
```

…is unambiguous.

<td>

```js
object.method(input);
```

<tr>
<td>

This:

```js
input |> object.method(); 🚫
```

…is invalid. It is invalid because it is
it does not have a topic reference.

<td>

The writing developer is forced by the compiler to clarify their intended
meaning, using a topic reference:

```js
input |> object.method(#);
input |> object.method()(#);
```

The reading developer benefits from explicitness and clarity, without
sacrificing the benefits of [untangled flow][] that pipelines bring.

<tr>
<td>

Adding other arguments:

```js
input |> object.method(x, y); 🚫
```

…is the same. This is an invalid pipeline.

<td>

The writer must clarify which of these reasonable interpretations is correct:

```js
input |> object.method(#, x, y);
input |> object.method(x, y, #);
input |> object.method(x, y)(#);
```

Both inserting the input as the first argument and inserting it as the last
argument are reasonable interpretations, as evidenced by how [other programming
languages’ pipe operators variously do either][topic references in other programming languages]. Or it could be a factory method that creates a function
that is in turn to be called with a unary input argument.

<tr>
<td>

Invalid topic-style pipeline:

```js
input |> await object.method(x, y); 🚫
```

<td>

Valid topic-style pipelines:

```js
input |> await object.method(#, x, y);
input |> await object.method(x, y, #);
input |> await object.method(x, y)(#);
input |> (await object.method(x, y))(#);
```

</table>

### Expressive versatility

JavaScript is a language rich with [expressions of numerous kinds][mdn operator precedence], each of which may usefully transform data from one
form to another. There is **no single type** of expression that forms a
**majority of used expressions**.

<table>
<tr>
<td>

1.  `undefined` and `null`.
2.  Boolean literals.
3.  String literals.
4.  Regular-expression literals.

<td>

5.  Template literals.
6.  Array literals.
7.  Object literals.

<tr>
<td>

8.  Variable references.
9.  Property accessors.
10. `this`.
11. `new.target`.

<td>

13. Arithmetic operations.
14. Bitwise operations.
15. Logical operations.

<tr>
<td>

16. Equality operations.
17. `instanceof` and `in` operations.
18. Conditional operations.

<td>

19. Unary function calls.
20. Unary constructor calls.
21. N-ary function calls.
22. N-ary constructor calls.
23. `super` calls.

<tr>
<td>

23. Arrow functions.
24. Function definitions.
25. Generator definitions.

<td>

26. Async-function definitions.
27. Async-generator definitions.
28. Class definitions.

<tr>
<td>

29. `typeof` operations.
30. `void` expressions.
31. `await` expressions.
32. `yield` expressions.

<td>

33. [Function binding?][function binding]

</table>

The goal of the pipe operator is to untangle deeply nested expressions into flat
threads of postfix expressions. To limit it to only one type of expression, even
a common type, truncates its benefits to that one type only and compromises its
expressivity and versatility.

In particular, relying on immediately invoked function expressions ([IIFEs][])
to accomodate non-unary function is insufficient for idiomatic JavaScript code.
JavaScript functions have never fulfilled the [Tennent correspondence
principle][]. Several common types of expressions cannot be equivalently used
within inner functions, particularly `await` and `yield`. In these frequent
cases, attempting to replacing code with “equivalent” IIFEs may cause different
behavior, may cause different performance behavior (see example in [zero runtime
cost][]), or may require dramatic rearrangement of logic to conserve the old
code’s behavior.

It would be possible to add ad-hoc handling, for selected other expression
types, to the operator’s grammar. This would expand its benefits to that type.
However, this conflicts with the goal of [cyclomatic simplicity][], by adding
complexity to the parsing process, proportional to the number of ad-hoc handled
cases. It also does not fulfill this goal well either: excluding, perhaps
arbitrarily, whatever classes its grammar’s branches do not handle.

Such new [incidental complexity][] makes code less readable and distracts the
developer from the program’s [essential logic][essential complexity]. A pipe
operator that improves readability should be versatile (this goal) but
[conceptually and cyclomatically simple][cyclomatic simplicity]. Such an
operator should be able to handle **all** expressions, in a **single** manner
**uniformly** **universally** applicable to **all** expressions. It is the hope
of this proposal’s authors that its [smart step syntax][] fulfills both criteria.

### Cyclomatic simplicity

Each edge case of the grammar increases the [cyclomatic complexity][] of parsing
the new syntax, increasing cognitive burden on both machine compiler and human
reader in writing and reading code without error. If edge cases and branching
are minimized, then the resulting syntax will be uniform and consistent. The
reduced complexity would hopefully reduce the probability that the developer
will misunderstand the code they read or write.

Similarly, reducing edge cases reduces the amount of trivia that a developer
must learn and remember in order to use the syntax. The more uniform and
simple the syntax’s rules, the more the developer may focus on the actual
meaning of their code.

Both [expressive versatility][] and simplicity are important components of
[“don’t make me overthink”][], but they sometimes conflict with one another.
When this happens, expressive versatility often wins: simplicity is important,
but sometimes it may be traded off for increased expressiveness. For instance,
[terse function calls][] are important for [tacit functional programming][tacit programming], one of the impetuses for the [first pipe-operator proposal][].

Supporting a [bare style](https://github.com/js-choi/proposal-smart-pipelines) brings
little expressive benefits, mainly it saves 3 characters.

## “Make my code easier to read.”

The new syntax should increase the human readability and writability of much
common code. It should be simpler to read and comprehend. And it should be
easier to compose and update. Otherwise, the new syntax would be useless.

Making JavaScript expressions more ergonomic for humans is the prime, original
purpose of this proposal. To a computer, the form of complex expressions –
whether as deeply nested groups or as flat threads of postfix steps – should not
matter. But to a human, it can make a significant difference.

### Untangled flow

When a human reads deeply nested groups of expressions – which are very common
in JavaScript code – their attention must switch between the start and end of
each nested expression. And these expressions will dramatically differ in
length, depending on their level in the syntactic tree. To use the example above:

```js
console.log(
  await stream.write(
    new User.Message(
      capitalize(
        doubledSay(
          (await promise) ||
            throw new TypeError(`Invalid value from ${promise}`)
        )
      ) + "!"
    )
  )
);
```

…the deep inner expression `await promise` is relatively short. In
contrast, the shallow outer expression
`` capitalize(doubledSay((await promise) || throw new TypeError(`Invalid value from ${promise}`))) + '!'`) ``
is very long. Yet both are
quite similar: they are transformations of a string into another. This
insight is lost in the deeply nested noise.

With pipelines, the code forms a flat thread of postfix steps. It is much
easier for a human to read and comprehend. Each of its steps are roughly the
same length. In order to understand what occurs before a given step, one
only need to scan left, rather than in both directions as the deeply nested
tree would require. To read the whole thing, a reader may simply follow
along left to right, not back and forth.

```js
promise
|> await #
|> # || throw new TypeError()
|> doubleSay(#, ', ')
|> capitalize(#)
|> # + '!'
|> new User.Message(#)
|> await stream.write(#)
|> console.log(#);
```

The introduction to this [motivation][] section already explained much of
the readability rationale.

### Distinguishable punctuators

Another important aspect of code readability is the visual distinguishability of
its most important words or symbols. Visually similar punctuators can distract
or even mislead the human reader, as they attempt to figure out the true meaning
of their code.

Any new punctuator should be easily distinguishable from existing symbols and should
not be visually confusable with unrelated syntax. This is particularly true for
choosing the topic-reference token, which would appear often in a wide variety
of expressions. If the topic reference hypothetically were `?` (and `??` and
`???` with [Additional Feature NP][]), and if the topic reference were used
anywhere near the visually similar [optional-chaining syntax proposal][] and
[nullish coalescing proposal][], then the topic reference might be lost or
unnoticed by the developer: for example, `(?)??.m(??)` is much less readable
than `#??.m(#)`.

### Terse parentheses

Terseness also aids distinguishability by obviating the need for boilerplate
syntactic noise. Parentheses are a prominent example: as long as operator
precedence is clear, then reducing parentheses always would JavaScript code more
visually terse and less cluttered.

The example above demonstrates how numerous verbose parentheses could become
unnecessary with pipelines. In these cases the [“data-to-ink” visual ratio][]
would significantly increase, emphasizing the program’s essential information.
The developer’s cognitive burden – of ignoring unimportant incidental symbols as
they read – has hopefully lightened.

### Terse variables

Similarly, terseness of code may also be increased by removing variables where
possible. This in turn would increase the data-to-ink visual ratio of the text
and the distinguishability of important symbols. This style of programming is
known as [tacit or point-free programming][tacit programming] (where “point”
refers to function arguments). Jeremy Gibbons, a computer scientist, expressed
its claimed benefits in a 1970 paper as such:

> Our calculations got completely bogged down using [function arguments]. In
> attempting to rephrase [function] definitions […] in particular, eliminating
> as many variables as possible and performing point-free (or ‘pointless‘)
> calculations at the level of functino compisition instead of point-wise
> calculations at the level of application, suddenly the calculations became
> almost trivial. This is the point of [point-free] calculations: when you
> travel light – discarding variables that do not contribute to the calculation
> – you can sometimes step lightly across the surface of the quagmire.

This sort of terseness, in which the explicit is made tacit and implicit, must
be balanced with [syntactic locality][] and [cyclomatic simplicity][]. Excessive
implicitness compromises comprehensibility, at least without low-level tracing
of tacit arguments’ invisible paths, rather than the actual, high-level meaning
of the code. Yet at the same time, excessive explicitness generates ritual,
verbose boilerplate that also interferes with reading comprehension. Therefore,
[untangled flow][] must be balanced with [backward compatibility][], [syntactic
locality][], and [cyclomatic simplicity][].

[The Zen of Python][pep 20] famously says, “Explicit is better than implicit,”
but it also says, “Flat is better than nested,” and, “Sparse is better than
dense.”

It's crucial that in this proposal the topic reference is always required, and what is substituded for it is always visible immediately on the left-hand side of the pipeline operator - in this way it avoids the pitfalls of tacit programming.

### Terse composition

Terse composition of all expressions – [not only unary functions][expressive versatility] but also n-ary functions, object methods, async functions,
generators, `if` `else` statements, and so forth – is a goal of pipelines.

## Other Goals

Although these have been prioritized last, they are still important.

### Human writability

Writability of code is less important a priority than readability of code. Code
is usually written a few days, perhaps by a few authors – but code will be read
dozens or hundreds of times, perhaps by many more people. However, ease of
writing and editing is still a good goal, and it often naturally increases when
code also becomes more readable. A useful heuristic for writability is assessing
the probability that a single edit to one piece of code will necessitate changes
to other parts of code that are not directly related to the edit.

The simple addition or removal of a deeply nested expression may necessitate the
indentation, de-indentation, parenthetical grouping, and parenthetical
flattening of many lines of code; the tedium of these incidental changes is a
major factor in the general popularity of automatic code formatters.

Achieving [static analyzability][] therefore also improves the ease of composing
and editing code. By flattening deeply nested expression trees into single
threads of postfix steps, a step may be added oredited in isolation on a single
line, it may be rearranged up or down, it may be removed – all without affecting
the pipeline’s other steps in the lines above or below it.

### Novice learnability

Learnability of the syntax is a desirable goal: the more intuitive the syntax
is, the more rapidly it might be adopted by developers. However, learnability in
of itself is not more desirable than the [other goals above][goals]. Most
JavaScript developers would be novices to this syntax at most once, during which
the intuitiveness of the syntax will dominate their experience. But after that
honeymoon period, the syntax’s usability in workaday programming will instead
affect their reading and writing most.

So instead, readability, comprehensibility, locality, simplicity,
expressiveness, and terseness are prioritized first, where they would conflict
with learnability itself. However, a syntax that is simple but expressive – and,
most of all, readable – could well be easier to learn. Its up-front cost in
learning could be small, particularly in comparison to the large gains in
readability and comprehensibility that it might bring to code in general.

["data-to-ink" visual ratio]: https://www.darkhorseanalytics.com/blog/data-looks-better-naked
["don’t break my code"]: ./goals.md#dont-break-my-code
["don’t make me overthink"]: ./goals.md#dont-make-me-overthink
["don’t shoot me in the foot"]: ./goals.md#dont-shoot-me-in-the-foot
["make my code easier to read"]: ./goals.md#make-my-code-easier-to-read
[`??:`]: https://github.com/tc39/proposal-nullish-coalescing/pull/23
[`do` expression]: ./relations.md#do-expressions
[`do` expressions]: ./relations.md#do-expressions
[`for` iteration statements]: https://tc39.github.io/ecma262/#sec-iteration-statements
[`in` relational operator]: https://tc39.github.io/ecma262/#sec-relational-operators
[`match` expressions]: ./relations.md#pattern-matching
[`new.target`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new.target
[additional feature ba]: ./additional-feature-ba.md
[additional feature bc]: ./additional-feature-bc.md
[additional feature bp]: ./additional-feature-bp.md
[additional feature np]: ./additional-feature-np.md
[additional feature pf]: ./additional-feature-pf.md
[additional feature ts]: ./additional-feature-ts.md
[additional features]: ./readme.md#additional-features
[annevk]: https://github.com/annevk
[antecedent]: https://en.wikipedia.org/wiki/Antecedent_(grammar)
[arbitrary associativity]: ./goals.md#arbitrary-associativity
[asi]: https://tc39.github.io/ecma262/#sec-automatic-semicolon-insertion
[associative property]: https://en.wikipedia.org/wiki/Associative_property
[async pipeline functions]: ./additional-feature-pf.md
[babel plugin]: https://github.com/valtech-nyc/babel/
[babel update summary]: https://github.com/babel/proposals/issues/29#issuecomment-372828328
[background]: ./readme.md
[backward compatibility]: ./goals.md#backward-compatibility
[bare awaited function call]: ./core-syntax.md#bare-style
[bare constructor call]: ./core-syntax.md#bare-style
[bare function call]: ./core-syntax.md#bare-style
[bare style]: ./core-syntax.md#bare-style
[binding]: https://en.wikipedia.org/wiki/Binding_(linguistics)
[block parameters]: ./relations.md#block-parameters
[clojure compact function]: https://clojure.org/reference/reader#_dispatch
[clojure pipe]: https://clojuredocs.org/clojure.core/as-%3E
[completion records]: https://timothygu.me/es-howto/#completion-records-and-shorthands
[concatenative programming]: https://en.wikipedia.org/wiki/Concatenative_programming_language
[conceptual generality]: ./goals.md#conceptual-generality
[core proposal]: ./readme.md
[core-real-examples.md]: ./core-real-examples.md
[currying]: https://en.wikipedia.org/wiki/Currying
[cyclomatic complexity]: https://en.wikipedia.org/wiki/Cyclomatic_complexity#Applications
[cyclomatic simplicity]: ./goals.md#cyclomatic-simplicity
[dataflow programming]: https://en.wikipedia.org/wiki/Dataflow_programming
[distinguishable punctuators]: ./goals.md#distinguishable-punctuators
[don’t break my code]: ./goals.md#dont-break-my-code
[dsls]: https://en.wikipedia.org/wiki/Domain-specific_language
[early error rule]: ./goals.md#static-analyzability
[early error rules]: ./goals.md#static-analyzability
[early error]: ./goals.md#static-analyzability
[early errors]: ./goals.md#static-analyzability
[ecmascript _identifier name_]: https://tc39.github.io/ecma262/#prod-IdentifierName
[ecmascript _identifier reference_]: https://tc39.github.io/ecma262/#prod-IdentifierReference
[ecmascript _member expression_]: https://tc39.github.io/ecma262/#prod-MemberExpression
[ecmascript § the syntactic grammar]: https://tc39.github.io/ecma262/#sec-syntactic-grammar
[ecmascript `new` operator, § rs: evaluation]: https://tc39.github.io/ecma262/#sec-new-operator-runtime-semantics-evaluation
[ecmascript arrow functions, § ss: contains]: https://tc39.github.io/ecma262/#sec-arrow-function-definitions-static-semantics-contains
[ecmascript assignment-level expressions]: https://tc39.github.io/ecma262/#sec-assignment-operators
[ecmascript block parameters]: https://github.com/samuelgoto/proposal-block-params
[ecmascript blocks, § rs: block declaration instantiation]: https://tc39.github.io/ecma262/#sec-blockdeclarationinstantiation
[ecmascript blocks, § rs: evaluation]: https://tc39.github.io/ecma262/#sec-block-runtime-semantics-evaluation
[ecmascript declarative environment records]: https://tc39.github.io/ecma262/#sec-declarative-environment-records
[ecmascript function binding]: https://github.com/zenparsing/es-function-bind
[ecmascript function calls, § rs: evaluation]: https://tc39.github.io/ecma262/#sec-function-calls-runtime-semantics-evaluation
[ecmascript function environment records]: https://tc39.github.io/ecma262/#sec-function-environment-records
[ecmascript functions and classes § generator function definitions]: https://tc39.github.io/ecma262/#sec-generator-function-definitions
[ecmascript get this environment]: https://tc39.github.io/ecma262/#sec-getthisenvironment
[ecmascript headless-arrow proposal]: https://bterlson.github.io/headless-arrows/
[ecmascript lexical environments]: https://tc39.github.io/ecma262/#sec-lexical-environments
[ecmascript lexical grammar]: https://tc39.github.io/ecma262/#sec-ecmascript-language-lexical-grammar
[ecmascript lhs expressions]: https://tc39.github.io/ecma262/#sec-left-hand-side-expressions
[ecmascript lists and records]: https://tc39.github.io/ecma262/#sec-list-and-record-specification-type
[ecmascript notational conventions, § algorithm conventions]: https://tc39.github.io/ecma262/#sec-algorithm-conventions-abstract-operations
[ecmascript notational conventions, § grammars]: https://tc39.github.io/ecma262/#sec-syntactic-and-lexical-grammars
[ecmascript notational conventions, § lexical grammar]: https://tc39.github.io/ecma262/#sec-lexical-and-regexp-grammars
[ecmascript notational conventions, § runtime semantics]: https://tc39.github.io/ecma262/#sec-runtime-semantics
[ecmascript optional `catch` binding]: https://github.com/tc39/proposal-optional-catch-binding
[ecmascript partial application]: https://github.com/tc39/proposal-partial-application
[ecmascript pattern matching]: https://github.com/tc39/proposal-pattern-matching
[ecmascript primary expressions]: https://tc39.github.io/ecma262/#prod-PrimaryExpression
[ecmascript property accessors, § rs: evaluation]: https://tc39.github.io/ecma262/#sec-property-accessors-runtime-semantics-evaluation
[ecmascript punctuators]: https://tc39.github.io/ecma262/#sec-punctuators
[ecmascript static semantic rules]: https://tc39.github.io/ecma262/#sec-static-semantic-rules
[elixir pipe]: https://hexdocs.pm/elixir/Kernel.html#%7C%3E/2
[elm pipe]: http://elm-lang.org/docs/syntax#infix-operators
[essential complexity]: https://en.wikipedia.org/wiki/Essential_complexity
[expressions and operators (mdn)]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Expressions_and_Operators
[expressive versatility]: ./goals.md#expressive-versatility
[f# pipe]: https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/functions/index#function-composition-and-pipelining
[first pipe-operator proposal]: https://github.com/tc39/proposal-pipeline-operator/blob/37119110d40226476f7af302a778bc981f606cee/README.md
[footguns]: https://en.wiktionary.org/wiki/footgun
[formal ba]: https://jschoi.org/18/es-smart-pipelines/spec#sec-additional-feature-ba
[formal bc]: https://jschoi.org/18/es-smart-pipelines/spec#sec-additional-feature-bc
[formal bp]: https://jschoi.org/18/es-smart-pipelines/spec#sec-additional-feature-bp
[formal cp]: https://jschoi.org/18/es-smart-pipelines/spec#introduction
[formal np]: https://jschoi.org/18/es-smart-pipelines/spec#sec-additional-feature-np
[formal pf]: https://jschoi.org/18/es-smart-pipelines/spec#sec-additional-feature-pf
[formal pipeline specification]: https://jschoi.org/18/es-smart-pipelines/spec
[formal ts]: https://jschoi.org/18/es-smart-pipelines/spec#sec-additional-feature-ts
[forward compatibility]: ./goals.md#forward-compatibility
[forward compatible]: ./goals.md#forward-compatibility
[function bind operator `::`]: ./relations.md#function-bind-operator
[function binding]: ./relations.md#function-binding
[function composition]: ./relations.md#function-composition
[functional programming]: https://en.wikipedia.org/wiki/Functional_programming
[gajus functional composition]: https://github.com/gajus/babel-plugin-transform-function-composition
[garden-path syntax]: https://en.wikipedia.org/wiki/Garden_path_sentence
[github issue tracker]: https://github.com/tc39/proposal-pipeline-operator/issues
[goals]: ./goals.md
[hack pipe]: https://docs.hhvm.com/hack/operators/pipe-operator
[huffman coding]: https://en.wikipedia.org/wiki/Huffman_coding
[human writability]: ./goals.md#human-writability
[i-am-tom functional composition]: https://github.com/fantasyland/ECMAScript-proposals/issues/1#issuecomment-306243513
[identity function]: https://en.wikipedia.org/wiki/Identity_function
[iifes]: https://en.wikipedia.org/wiki/Immediately-invoked_function_expression
[immutable objects]: https://en.wikipedia.org/wiki/Immutable_object
[incidental complexity]: https://en.wikipedia.org/wiki/Incidental_complexity
[intro]: readme.md
[isiahmeadows functional composition]: https://github.com/isiahmeadows/function-composition-proposal
[jashkenas]: https://github.com/jashkenas
[jquery + cp]: ./core-real-examples.md#jquery
[jquery]: https://jquery.com/
[jquery/src/core/access.js]: https://github.com/jquery/jquery/blob/2.2-stable/src/core/access.js
[jquery/src/core/init.js]: https://github.com/jquery/jquery/blob/2.2-stable/src/core/init.js
[jquery/src/core/parsehtml.js]: https://github.com/jquery/jquery/blob/2.2-stable/src/core/parseHTML.js
[js foundation]: https://js.foundation/
[julia pipe]: https://docs.julialang.org/en/stable/stdlib/base/#Base.:|>
[lexical topic]: https://jschoi.org/18/es-smart-pipelines/spec#sec-lexical-topics
[lexically hygienic]: https://en.wikipedia.org/wiki/Hygienic_macro
[littledan invitation]: https://github.com/tc39/proposal-pipeline-operator/issues/89#issuecomment-363853394
[littledan]: https://github.com/littledan
[livescript pipe]: http://livescript.net/#operators-piping
[lodash + cp + bp + pf]: ./additional-feature-pf.md#lodash-core-proposal--additional-feature-bppf
[lodash + cp + bp + pp + pf + np]: ./additional-feature-np.md#lodash-core-proposal--additional-features-bppppfmt
[lodash + cp]: ./core-real-examples.md#lodash
[lodash]: https://lodash.com/
[maadhattah]: https://github.com/mAAdhaTTah/
[make my code easier to read]: ./goals.md#make-my-code-easier-to-read
[mdn operator precedence]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence
[mdn operator precedence]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence#Table
[method-extraction inline caching]: https://github.com/tc39/proposal-bind-operator/issues/46
[mindeavor]: https://github.com/gilbert
[mode errors]: https://en.wikipedia.org/wiki/Mode_(computer_interface)#Mode_errors
[motivation]: ./readme.md
[node-stream piping]: https://nodejs.org/api/stream.html#stream_readable_pipe_destination_options
[node.js `util.promisify`]: https://nodejs.org/api/util.html#util_util_promisify_original
[nomenclature]: ./nomenclature.md
[novice learnability]: ./goals.md#novice-learnability
[nullish coalescing proposal]: https://github.com/tc39/proposal-nullish-coalescing/
[object initializers’ computed property contains rule]: https://tc39.github.io/ecma262/#sec-object-initializer-static-semantics-computedpropertycontains
[ocaml pipe]: http://blog.shaynefletcher.org/2013/12/pipelining-with-operator-in-ocaml.html
[operator precedence]: ./core-syntax.md#operator-precedence-and-associativity
[opt-in behavior]: ./goals.md#opt-in-behavior
[optional `catch` binding]: ./relations.md#optional-catch-binding
[optional-chaining syntax proposal]: https://github.com/tc39/proposal-optional-chaining
[other browsers’ console variables]: https://www.andismith.com/blogs/2011/11/25-dev-tool-secrets/
[other ecmascript proposals]: ./relations.md#other-ecmascript-proposals
[other goals]: ./goals.md#other-goals
[pattern matching]: ./relations.md#pattern-matching
[partial function application]: ./goals.md#partial-function-application
[pep 20]: https://www.python.org/dev/peps/pep-0020/
[perl 6 pipe]: https://docs.perl6.org/language/operators#infix_==>
[perl 6 topicization]: https://www.perl.com/pub/2002/10/30/topic.html/
[perl 6’s given block]: https://docs.perl6.org/language/control#given
[pipeline functions]: ./additional-feature-pf.md
[pipeline proposal 1]: https://github.com/tc39/proposal-pipeline-operator/wiki#proposal-1-f-sharp-only
[pipeline proposal 4]: https://github.com/tc39/proposal-pipeline-operator/wiki#proposal-4-smart-mix
[pipeline syntax]: ./core-syntax.md
[pipelines]: ./readme.md
[previous pipeline-placeholder discussions]: https://github.com/tc39/proposal-pipeline-operator/issues?q=placeholder
[primary topic]: ./readme.md
[private class fields]: https://github.com/tc39/proposal-class-fields/
[pure functions]: https://en.wikipedia.org/wiki/Pure_function
[r pipe]: https://cran.r-project.org/web/packages/magrittr/vignettes/magrittr.html
[ramda + cp + bp + pf + np]: ./additional-feature-np.md#ramda-core-proposal--additional-features-bppfmt
[ramda + cp + bp + pf]: ./additional-feature-pf.md#ramda-core-proposal--additional-feature-bppf
[ramda wiki cookbook]: https://github.com/ramda/ramda/wiki/Cookbook
[ramda]: http://ramdajs.com/
[relations to other work]: ./relations.md
[repls]: https://en.wikipedia.org/wiki/Read–eval–print_loop
[rest topic]: ./additional-feature-np.md
[reverse polish notation]: https://en.wikipedia.org/wiki/Reverse_Polish_notation
[robust method extraction]: https://github.com/tc39/proposal-pipeline-operator/issues/110#issuecomment-374367888
[ron buckton]: https://github.com/rbuckton
[secondary topic]: ./additional-feature-np.md
[semantic clarity]: ./goals.md#semantic-clarity
[simonstaton functional composition]: https://github.com/simonstaton/Function.prototype.compose-TC39-Proposal
[simple scoping]: ./goals.md#simple-scoping
[sindresorhus]: https://github.com/sindresorhus
[smart step syntax]: ./core-syntax.md
[smart pipelines]: ./readme.md#smart-pipelines
[standard style]: https://standardjs.com/
[static analyzability]: ./goals.md#static-analyzability
[statically analyzable]: ./goals.md#static-analyzability
[statically detectable early errors]: ./goals.md#static-analyzability
[syntactic locality]: ./goals.md#syntactic-locality
[tacit programming]: https://en.wikipedia.org/wiki/Tacit_programming
[tc39 60th meeting, pipelines]: https://tc39.github.io/tc39-notes/2017-09_sep-26.html#11iia-pipeline-operator
[tc39 process]: https://tc39.github.io/process-document/
[tc39/proposal-decorators#30]: tc39/proposal-decorators#30
[tc39/proposal-decorators#42]: tc39/proposal-decorators#42
[tc39/proposal-decorators#60]: tc39/proposal-decorators#60
[tennent correspondence principle]: http://gafter.blogspot.com/2006/08/tennents-correspondence-principle-and.html
[term rewriting]: ./term-rewriting.md
[terse composition]: ./goals.md#terse-composition
[terse function application]: ./goals.md#terse-function-application
[terse function calls]: ./goals.md#terse-function-calls
[terse method extraction]: ./goals.md#terse-method-extraction
[terse parentheses]: ./goals.md#terse-parentheses
[terse partial application]: ./goals.md#terse-partial-application
[terse variables]: ./goals.md#terse-variables
[tertiary topic]: ./additional-feature-np.md
[thenavigateur functional composition]: https://github.com/TheNavigateur/proposal-pipeline-operator-for-function-composition
[topic and comment]: https://en.wikipedia.org/wiki/Topic_and_comment
[topic references in other programming languages]: ./relations.md#topic-references-in-other-programming-languages
[topic style]: ./core-syntax.md#topic-style
[topic variables in other languages]: https://rosettacode.org/wiki/Topic_variable
[topic-token bikeshedding]: https://github.com/tc39/proposal-pipeline-operator/issues/91
[underscore.js + cp + bp + pp]: ./additional-feature-pp.md#underscorejs-core-proposal--additional-feature-bppp
[underscore.js + cp]: ./core-real-examples.md#underscorejs
[underscore.js]: http://underscorejs.org
[unix pipe]: https://en.wikipedia.org/wiki/Pipeline_(Unix
[untangled flow]: ./goals.md#untangled-flow
[visual basic’s `select` statement]: https://docs.microsoft.com/en-us/dotnet/visual-basic/language-reference/statements/select-case-statement
[webkit console variables]: https://webkit.org/blog/829/web-inspector-updates/
[whatwg fetch + cp]: ./core-real-examples.md#whatwg-fetch-standard
[whatwg fetch standard]: https://fetch.spec.whatwg.org/
[whatwg streams + cp + bp + pf + np]: ./additional-feature-np.md#whatwg-streams-standard-core-proposal--additional-features-bppfmt
[whatwg streams + cp + bp + pf]: ./additional-feature-pf.md#whatwg-streams-standard-core-proposal--additional-feature-bppf
[whatwg streams standard]: https://stream.spec.whatwg.org/
[whatwg-stream piping]: https://streams.spec.whatwg.org/#pipe-chains
[wikipedia: term rewriting]: https://en.wikipedia.org/wiki/Term_rewriting
[zero runtime cost]: ./goals.md#zero-runtime-cost
