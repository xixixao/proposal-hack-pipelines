# Smart pipelines

ECMAScript Stage-0 Proposal. Living Document. Michal Srb, 2020-06.

> Note: This is a fork of the amazing work done by J. S. Choi on the ["Smart" pipeline proposal](https://github.com/js-choi/proposal-smart-pipelines). This explainer and spec are stripped down of the "bare" style and motivation to support truly tacit programming. In turn this is a [much simpler proposal][why-not-smart].

This proposal introduces a new binary pipe operator `|>` to JavaScript. It's similar
to the pipe operators of other languages: Clojure, Elixir and Erlang,
Elm, F#, Hack, Julia, LiveScript, OCaml, Perl 6, R with magrittr, and Unix shells and PowerShell.

```js
value
  |> await #
  |> doubleSay(#, ', ')
  |> capitalize(#)
  |> # + '!'
  |> new User.Message(#)
  |> await #
  |> console.log(#);
```

The proposal is currently at **Stage 0** of the [TC39 process][tc39 process] and is planned to be presented. There are competing proposals, see this [repo](https://github.com/tc39/proposal-pipeline-operator).

There's a [PR](https://github.com/babel/babel/pull/11600) to add support for this version into Babel, but if you just want to get a feel for it use the "smart" proposal implementation but don't use the "bare style".

You can take part in discussions on the original proposal's **[GitHub issue tracker][]**.
When you file an issue, please note in it that you are talking **specifically** about
**[“Proposal 2: Hack Style Only”][pipeline proposal 2]**.

<table>
<thead>
<tr>
<th>With pipelines in JS
<th>With no pipelines

<tbody>
<tr>
<td>

The infix “smart” pipe operator `|>` proposed here would provide a
**[backwards- and forwards-compatible][don’t break my code]** style of
**chaining nested expressions** into a readable, **left-to-right** manner.

Using a **[zero-cost abstraction][zero runtime cost]**, **nested** data
transformations become [**untangled** into **short steps**][untangled flow].

<td>

**Nested, deeply composed** expressions occur often in [real-world JavaScript][core-real-examples.md].
They occur whenever any single value must be processed by a **series of data
transformations**, whether they be **operations, functions, or constructors**.
Unfortunately, these deeply nested expressions often result in **messy
spaghetti** code, due to their mixing of **prefix, infix, and postfix**
expressions together. Writing such code requires many nested **levels of
indentation** and parentheses. Reading such code requires checking **both the
left and right of each subexpression** to understand its data flow.

<tr>
<td>

```js
promise
  |> await #
  |> # || throw new TypeError(
    `Invalid value from ${promise}`)
  |> doubleSay(#, ', ')
  |> capitalize(#)
  |> # + '!'
  |> new User.Message(#)
  |> await stream.write(#)
  |> console.log(#);
```

With pipelines, code becomes more **straightforward**.
Prefix, infix, and postfix expressions would be less
tangled together in threads of spaghetti. Instead, data values would be **piped
from left to right** through a **single linear thread of postfix expressions**,
with a [**single** level of **indentation**][untangled flow] and [**four fewer**
pairs of **parentheses**][terse parentheses] – essentially forming a [reverse
Polish notation][].

The resulting code’s [**terseness** and **linearity**][make my code easier to read] may be both easier for the JavaScript developer to **read** and to
**edit**. This uniform postfix notation preserves locality between related code;
the reader may **follow the flow** of data more easily through this [single
linear thread of postfix operations][untangled flow]. And the developer may
[more easily **add or remove operations**][human writability] at the beginning,
end, or middle of the thread, **without changing** the **indentation** of
unrelated lines.

<td>

```js
console.log(
  await stream.write(
    new User.Message(
      capitalize(
        doubledSay(
          (await promise) ||
            throw new TypeError(`Invalid value from ${promise}`)
        ),
        ", "
      ) + "!"
    )
  )
);
```

Compared with the pipeline version, this non-pipeline version requires **additional
indentation and grouping** on each **step**. This requires four more levels of
indentation and four more pairs of parentheses.

In addition, much related code is here separated by unrelated code. Rather than
a **uniform** postfix chain, operations appear **either before** the previous
**step**’s expression (`await stream.write(…)`,`new User.Message(…)`,
`capitalize(…)`, `doubledSay(…)`, `await …`) but also **after** (`… || throw new TypeError()`, `… + '!'`). An additional argument to function calls (such as
`,` in `doubledSay(…, ', ')`) is also separated from its function calls,
forming another easy-to-miss “postfix” argument.

<tr>
<td>

Each **step** of a pipeline creates its own lexical scope, within which the `#`
token (the **topic reference**) is immutably bound to the result of the previous
step (the **topic**). In this way, it acts as a placeholder for each step’s input.

```js
input // step 0
  |> # + 1 // step 1
  |> f(x, #, y) // step 2
  |> await g(#, z) // step 3
  |> console.log(`${#}!`); // step 4
```

<td>

```js
console.log(
  `${
    // step 4
    await g(
      // step 3
      f(
        x, // step 2
        input + 1, // step 1
        y
      ), // step 2
      z
    ) // step 3
  }!`
); // step 4
```

<tr>
<td>

```js
input |> (# = 50);
// 🚫 Reference Error:
// Cannot assign to topic reference.
```

The topic binding is immutable, established only once per pipeline step.
It is a syntax error to attempt to assign a value to it using `=`, whether inside or outside a pipeline step.

<td>

<tr>
<td>

This chained pipeline:

```js
input
  |> # - 3
  |> -#
  |> # * 2
  |> Math.max(#, 0)
  |> console.log(#);
```

<td>

…is equivalent to the tangled nested expression:

```js
console.log(Math.max(-(input - 3) * 2, 0));
```

<tr>
<td>

The syntax is [**statically term rewritable** into already valid code][term rewriting] in this way, with [theoretically **zero runtime cost**][zero runtime cost].

Similar use cases appear **numerous times** in [real-world JavaScript code][core-real-examples.md],
whenever any input is transformed by **[expressions of any type][expressive versatility]**:
function calls, property calls, method calls, object constructions, arithmetic
operations, logical operations, bitwise operations, `typeof`, `instanceof`,
`await`, `yield` and `yield *`, and `throw` expressions.

<tr>
<td>

```js
function doubleSay (str, separator) {
  return `${str}${separator}${string}`;
}

function capitalize (str) {
  return str[0].toUpperCase()
    + str.substring(1);
}

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

This pipeline is also relatively linear, with only one level of indentation, and
with each transformation step on its own line.

`… |> capitalize` uses a special **shortcut** called the **[bare style][]**, explained further below.
It is a bare unary function call that is, in this case, equivalent to `… |> capitalize(#)`.

<td>

```js
function doubleSay(str, separator) {
  return `${str}${separator}${str}`;
}

function capitalize(str) {
  return str[0].toUpperCase() + str.substring(1);
}

console.log(
  await stream.write(
    new User.Message(
      capitalize(
        doubledSay(
          (await promise) ||
            throw new TypeError(`Invalid value from ${promise}`)
        ),
        ", "
      ) + "!"
    )
  )
);
```

This deeply nested expression has four levels of indentation instead of two.
Reading its data flow requires checking both the beginning of each expression
(`new User.Message`, `capitalizedString`, `doubledSay`, `await promise` and
end of each expression (`|| throw new TypeError()`, `, ', '`, `+ '!'`)).

<tr>
<td>

```js
x = … |> f(#, #);
```

```js
x = … |> [#, # * 2, # * 3];
```

The topic reference may be used multiple times in a pipeline step. Each use
refers to the same value (wherever the topic reference is not overridden by
another, inner pipeline’s topic scope). Because it is bound to the result of the
topic, the topic is still only ever evaluated once.

<td>

```js
{
  const $ = …;
  x = f($, $);
}
```

```js
{
  const $ = …;
  x = [$, $ * 2, $ * 3];
}
```

This is equivalent to assigning the topic value to a [unique variable][lexically hygienic], then using that variable multiple times in an expression.

<tr>
<td>

```js
promise
  |> await #
  |> # ?? throw new TypeError()
  |> `${#}, ${#}`
  |> #[0].toUpperCase() + #.substring(1)
  |> # + '!'
  |> new User.Message(#)
  |> stream.write(#)
  |> console.log(#);
```

When tiny functions are only used once, and when their bodies would be obvious and
self-documenting in meaning, then they might be ritual boilerplate that a developer
may prefer to inline: trading off self-documentation for localization of code.

<td>

```js
{
  const promiseValue = (await promise) ?? throw new TypeError();
  const doubledValue = `${promiseValue}, ${promiseValue}`;
  const capitalizedValue =
    doubledValue[0].toUpperCase() + doubledValue.substring(1);
  const exclaimedValue = capitalizedValue + "!";
  const userMessage = new User.Message(exclaimedValue);
  const writeValue = stream.write(userMessage);
  console.log(writeValue);
}
```

Using a sequence of variables instead has both advantages and disadvantages. The
variable names may be self-documenting. But they also are verbose. They visually
distract from the crucial data transformations (overemphasizing the expressions’
nouns over their verbs), and they can be mistyped.

<tr>
<td>

```js
  promise
  |> await #
  |> # ?? throw new TypeError()
  |> normalize(#)
  |> `${#}, ${#}`
  |> #[0].toUpperCase() + #.substring(1)
  |> # + '!'
  |> new User.Message(#)
  |> await stream.write(#)
  |> console.log(#);
```

With a pipeline, there are no unnecessary variable identifiers. Inserting a new
step in between two steps (or deleting a step) only touches one new line. Here,
a call of a function `normalize` was inserted between the second and third steps.

<td>

```js
{
  const promiseValue = (await promise) ?? throw new TypeError();
  const normalizedValue = normalize();
  const doubledValue = `${normalizedValue}, ${normalizedValue}`;
  const capitalizedValue =
    doubledValue[0].toUpperCase() + doubledValue.substring(1);
  const exclaimedValue = capitalizedValue + "!";
  const userMessage = new User.Message(exclaimedValue);
  const writeValue = stream.write(userMessage);
  console.log(writeValue);
}
```

This code underwent a similar insertion of `normalize`. With a series of
variables, inserting a new step in between two other steps (or deleting a step)
requires editing the variable names in the following step.

<tr>
<td>

```js
input |> f(#) |> [0, 1, 2, ...#] |> g(#);
```

A pipeline step may contain array literals.

<td>

```js
g([0, 1, 2, ...f(input)]);
```

<tr>
<td>

A pipeline step may also contain object literals. However, pipeline
steps that are entirely object literals must be parenthesized. It is similar to
how arrow functions distinguish between object literals and blocks.

```js
input |> f(#) |> ({ x: #, y: # }) |> g(#);
```

This is for [forward compatibility][] with potentially supporting block pipeline steps. (It is expected that block pipelines would
eventually be much more common than pipelines with object literals.) This restriction
could be dropped for simplicity, or in favor of support for `do` expressions.

```js
input |> f(#) |> { x: #, y: # } |> g(#);
// 🚫 Syntax Error:
// Unexpected token `{`.
// Cannot parse base expression.
```

<td>

```js
{
  const $ = f(input);
  g({ x, $: y: f($) });
}
```

<tr>
<td>

```js
return input
  |> f(#)
  |> (x) => # + x;
```

A pipeline step may contain an inner arrow function. Both versions
of this example result in an arrow function in a closure on the previous
pipeline’s result `input |> f`.

<td>

```js
const $ = f(input);
return (x) => $ + x;
```

The arrow function lexically closes over the topic value, takes one parameter,
and returns the sum of the topic value and the parameter.

<tr>
<td>

```js
input
  |> f(#)
  |> settimeout(() => # * 5)
  |> processIntervalID(#);
```

This ability to create arrow functions, which do not lexically shadow the topic,
can be useful for using callbacks in a pipeline.

<td>

```js
{
  const $ = f(input);
  const intervalID = settimeout(() => $ * 5);
  processIntervalID(intervalID);
}
```

The topic value of the second pipeline step (here represented by a normal
variable `$`) is still lexically accessible within its body, an arrow function,
in both examples.

<tr>
<td>

```js
input
|> f
|> () => # * 5
|> settimeout(#)
|> processIntervalID(#);
```

The arrow function can also be created on a separate pipeline step.

<td>

```js
{
  const $ = f(input);
  const callback = () => $ * 5;
  const intervalID = settimeout(callback);
  processIntervalID(intervalID);
}
```

The result here is the same.

<tr>
<td>

```js
input |> x + 50 |> f |> g(x, 2);
// 🚫 Syntax Error:
// Topic-style pipeline step
// `|> x + 50`
// binds topic but contains
// no topic reference.
// 🚫 Syntax Error:
// Topic-style pipeline step
// `|> g(x, 2)`
// binds topic but contains
// no topic reference.
```

In order to fulfill the [goal][goals] of [“don’t shoot me in the foot”][],
when a pipeline step it **contains no topic
reference**, that is an **[early error][]**. Such a degenerate
pipeline step has a very good chance of actually being an accidental bug.

<td>

<tr>
<td>

Currently, **four kinds of statements cannot** use an **outside context’s topic**
in their expressions. These are:

- `function` definitions (including those for async functions, generators, and
  async generators; but **not** arrow functions, as explained above),
- `class` definitions,
- `for` and `while` statements,
- `catch` clauses (but see [Additional Feature TS][]), and
- `with` statements.

```js
x = input |> function () { return #; };
// 🚫 Syntax Error:
// Lexical context `function () { return #; }`
// contains a topic reference
// but has no topic binding.
// 🚫 Syntax Error:
// Pipeline step `|> function () { … }`
// binds topic but contains
// no topic reference.
```

This behavior is in order to fulfill the [goals][] of [simple scoping][] and of
[“don’t shoot me in the foot”][]: it prevents the origin of any topic from being
difficult to find. It also fulfills the goal of [forward compatibility][].

However, this behavior is subject to change.

<td>

<tr>
<td>

```js
x = input |> class { m: () { return #; } };
// 🚫 Syntax Error:
// Pipeline step `|> class { … }`
// binds topic but contains
// no topic reference.
```

<td>

<tr>
<td>

```js
x = input
  |> await f(#, 5)
  |> () => {
    if (#)
      return # + 30;
    else
      return #;
  }
  |> g(#);
```

Any other nested blocks **may** contain topic references from outer lexical
environments. These include **arrow functions, `if` statements**, `try`
statements and their `finally` clauses (though not their `catch` clauses),
`switch` statements, and bare block statements.

TODO: Consider changing this to further prevent people from abusing this feature.

<td>

```js
x = g((await f(input, 5)) + 30);
```

<tr>
<td>

A function definition that is a pipeline step may contain topic references in
its default parameters’ expressions, because their scoping is similar to that of
the outside context’s: similar enough such that also allowing topic references
in them would fulfill the goal of [simple scoping][]. However, as stated above,
the function body itself still may not contain topic references.

TODO: Consider changing this to further prevent people from abusing this feature.

```js
value
  |> processing
  |> function (x = #) { return x; }
```

<td>

```js
function (x = processing(value)) {
  return x;
}
```

<tr>
<td>

The same applies to the parenthesized antecedents of `for` and `while` loops.

```js
  input
  |> process
  |> (x, y) => {
    for (const element of #)
      …
  }
```

```js
input
  |> process
  |> (x, y) => {
    let element;
    while (element = getNextFrom(#))
      …
  }
```

<td>

```js
(x, y) => {
  for (const element
    of process(input))
    …
}
```

```js
(x, y) {
  let element;
  while (element =
    getNextFrom(input))
    …
}
```

</table>

## Real-world examples

See [core-real-examples.md][].

## Additional features

This document is an **explainer for** the [**formal specification** of a proposed
**pipe operator `|>`**][formal pipeline specification] in
**JavaScript**. The specification
is divided into **one Stage-0 Core Proposal** plus **one**
independent-but-compatible **additional feature** (TODO: This will be [optional pipes](https://github.com/tc39/proposal-pipeline-operator/issues/159)).

# Legacy link anchors

For resources can be found below:

## Core Proposal real-world examples

See [core-real-examples.md][].

### WHATWG Fetch Standard (Core Proposal only)

See [WHATWG Fetch + CP][].

### jQuery (Core Proposal only)

See [jQuery + CP][].

### Underscore.js (Core Proposal only)

See [Underscore.js + CP][].

### Lodash (Core Proposal only)

See [Lodash + CP][].

## Smart pipe style

See [core syntax](core-syntax.md).

### Bare style

See [core syntax, bare style][bare style].

### Topic style

See [core syntax, topic style][topic style].

## Additional Feature BC

See [Additional Feature BC][].

## Additional Feature BA

See [Additional Feature BA][].

## Additional Feature BP

See [Additional Feature BP][].

### WHATWG Fetch Standard (Core Proposal + Additional Feature BP)

See [Additional Feature BP][].

### jQuery (Core Proposal + Additional Feature BP)

See [Additional Feature BP][].

### Lodash (Core Proposal + Additional Feature BP)

See [Additional Feature BP][].

## Additional Feature TS

See [Additional Feature TS][].

## Additional Feature PF

See [Additional Feature PF][].

### Ramda (Core Proposal + Additional Feature BP+PF)

See [Additional Feature PF][].

### WHATWG Streams Standard (Core Proposal + Additional Features BP+PP+PF)

See [Additional Feature PF][].

## Additional Feature NP

See [Additional Feature NP][].

### Lodash (Core Proposal + Additional Features BP+PP+PF+NP)

See [Additional Feature NP][].

### Ramda (Core Proposal + Additional Features BP+PF+NP)

See [Additional Feature NP][].

### WHATWG Streams Standard (Core Proposal + Additional Features BP+PP+PF+NP)

See [Additional Feature NP][].

## Goals

See [Goals][].

### “Don’t break my code.”

See [“Don’t break my code”][].

#### Backward compatibility

See [Backward compatibility][].

#### Zero runtime cost

See [Zero runtime cost][].

#### Forward compatibility

See [Forward compatibility][].

### “Don’t shoot me in the foot.”

See [“Don’t shoot me in the foot”][].

#### Opt-in behavior

See [Opt-in behavior][].

#### Simple scoping

See [Simple scoping][].

#### Static analyzability

See [Static analyzability][].

### “Don’t make me overthink.”

See [“Don’t make me overthink”][].

#### Syntactic locality

See [Syntactic locality][].

#### Semantic clarity

See [Semantic clarity][].

#### Expressive versatility

See [Expressive versatility][].

#### Cyclomatic simplicity

See [Cyclomatic simplicity][].

### “Make my code easier to read.”

See [“Make my code easier to read”][].

#### Untangled flow

See [Untangled flow][].

#### Distinguishable punctuators

See [Distinguishable punctuators][].

#### Terse parentheses

See [Terse parentheses][].

#### Terse variables

See [Terse variables][].

#### Terse function calls

See [Terse function calls][].

#### Terse composition

See [Terse composition][].

#### Terse partial application

See [Terse partial application][].

### Other Goals

See [Other Goals][].

#### Conceptual generality

See [Conceptual generality][].

#### Human writability

See [Human writability][].

#### Novice learnability

See [Novice learnability][].

## Relations to other work

See [Relations to other work][].

### Pipelines in other programming languages

See [Relations to other work][].

### Topic references in other programming languages

See [Topic references in other programming languages][].

### `do` expressions

See [`do` expressions][].

### Function binding

See [Function binding][].

### Function composition

See [Function composition][].

### Partial function application

See [Partial function application][].

### Optional `catch` binding

See [Optional `catch` binding][].

### Pattern matching

See [Pattern matching][].

### Block parameters

See [Block parameters][].

### `do` expressions

See [`do` expressions][].

## Explanation of nomenclature

See [Nomenclature][].

## Term rewriting

See [Term rewriting][].

[“data-to-ink” visual ratio]: https://www.darkhorseanalytics.com/blog/data-looks-better-naked
[“don’t break my code”]: ./goals.md#dont-break-my-code
[“don’t make me overthink”]: ./goals.md#dont-make-me-overthink
[“don’t shoot me in the foot”]: ./goals.md#dont-shoot-me-in-the-foot
[“make my code easier to read”]: ./goals.md#make-my-code-easier-to-read
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
[formal pipeline specification]: https://xixixao.github.io/proposal-hack-pipelines
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
[pipeline proposal 2]: https://github.com/tc39/proposal-pipeline-operator/wiki#proposal-2-hack-style-only
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
[why-not-smart]: ./why-not-so-smart.md
[wikipedia: term rewriting]: https://en.wikipedia.org/wiki/Term_rewriting
[zero runtime cost]: ./goals.md#zero-runtime-cost
