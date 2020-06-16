# Core Proposal real-world examples

Living Document. Michal Srb, 2020-06.

> NOTE: The examples here are pretty weak and often complicated. We should be able to find better ones. Vast majority of normal JS code is function calls, not all kinds of expressions combined together.

### WHATWG Fetch Standard

The [WHATWG Fetch Standard][] contains several examples of using the DOM `fetch`
function, resolving its promises into values, then processing the values in
various ways. These examples may become more easily readable with smart pipelines.

<table>
<thead>
<tr>
<th>With smart pipelines
<th>Status quo

<tbody>
<tr>
<td>

```js
'/music/pk/altes-kamuffel'
  |> await fetch(#)
  |> await #.blob()
  |> playBlob(#);
```

<td>

```js
fetch("/music/pk/altes-kamuffel")
  .then((res) => res.blob())
  .then(playBlob);
```

<tr>
<td>
[Equivalent to above.]

<td>

```js
playBlob(await (await fetch("/music/pk/altes-kamuffel")).blob());
```

<tr>
<td>

```js
'https://example.com/'
  |> await fetch(#, { method: 'HEAD' })
  |> #.headers.get('content-type')
  |> console.log(#);
```

<td>

```js
fetch("https://example.com/", { method: "HEAD" }).then((response) =>
  console.log(response.headers.get("content-type"))
);
```

<tr>
<td>

```js
'https://example.com/'
  |> await fetch(#, { method: 'HEAD' })
  |> #.headers.get('content-type')
  |> console.log(#);
```

<td>

```js
console.log(
  (await fetch("https://example.com/", { method: "HEAD" })).headers.get(
    "content-type"
  )
);
```

<tr>
<td>
[Equivalent to above.]

<td>

```js
{
  const url = "https://example.com/";
  const response = await fetch(url, { method: "HEAD" });
  const contentType = response.headers.get("content-type");
  console.log(contentType);
}
```

</table>

### jQuery

As the single most-used JavaScript library in the world, [jQuery][] has provided
an alternative human-ergonomic API to the DOM since 2006. jQuery is under the
stewardship of the [JS Foundation][], a member organization of TC39 through which
jQuery’s developers are represented in TC39. jQuery’s API requires complex data
processing that becomes more readable with smart pipelines.

<table>
<thead>
<tr>
<th>With smart pipelines
<th>Status quo

<tbody>
<tr>
<td>

```js
return data
|> buildFragment([#], context, scripts)
|> #.childNodes
|> jQuery.merge([], #);
```

The path that a reader’s eyes must trace while reading this pipeline moves
straight down, with some movement toward the right then back: from `data` to
`buildFragment` (and its arguments), then `.childNodes`, then `jQuery.merge`.
Here, no one-off-variable assignment is necessary.

<td>

```js
parsed = buildFragment([data], context, scripts);
return jQuery.merge([], parsed.childNodes);
```

From [jquery/src/core/parseHTML.js][]. In this code, the eyes first must look
for `data` – then upwards to `parsed = buildFragment` (and then back for
`buildFragment`’s other arguments) – then down, searching for the location of
the `parsed` variable in the next statement – then right when noticing its
`.childNodes` postfix – then back upward to `return jQuery.merge`.

<tr>
<td>

```js
(key |> toType(#)) === 'object';
```

```js
key |> toType(#) |> # === 'object';
```

`|>` has a looser precedence than most operators, including `===`. (Only
assignment operators, arrow function `=>`, yield operators, and the comma
operator are any looser.)

<td>

```js
toType(key) === "object";
```

From [jquery/src/core/access.js][].

<tr>
<td>

```js
context = context
|> # instanceof jQuery
    ? #[0] : #;
```

<td>

```js
context = context instanceof jQuery ? context[0] : context;
```

From [jquery/src/core/access.js][].

<tr>
<td>

```js
context
|> # && #.nodeType
  ? #.ownerDocument || #
  : document
|> jQuery.parseHTML(match[1], #, true)
|> jQuery.merge(#);
```

<td>

```js
jQuery.merge(
  this,
  jQuery.parseHTML(
    match[1],
    context && context.nodeType ? context.ownerDocument || context : document,
    true
  )
);
```

From [jquery/src/core/init.js][].

<tr>
<td>

```js
match
|> context[#]
|> (this[match] |> isFunction(#))
  ? this[match](#);
  : this.attr(match, #);
```

Note how, in this version, the parallelism between the two clauses is very
clear: they both share the form `match |> context[#] |> something(match, #)`.

<td>

```js
if (isFunction(this[match])) {
  this[match](context[match]);
} else
  this.attr(match, context[match]);
}
```

From [jquery/src/core/init.js][]. Here, the parallelism between the clauses
is somewhat less clear: the common expression `context[match]` is at the end
of both clauses, at a different offset from the margin.

<tr>
<td>

```js
elem = match[2] |> document.getElementById;
```

<td>

```js
elem = document.getElementById(match[2]);
```

From [jquery/src/core/init.js][].

<tr>
<td>

```js
// Handle HTML strings
if (…)
  …
// Handle $(expr, $(...))
else if (!# || #.jquery)
  return context
    |> # || root
    |> #.find(selector);
// Handle $(expr, context)
else
  return context
    |> this.constructor
    |> #.find(selector);
```

The parallelism between the final two clauses becomes clearer here too.
They both are of the form `return context |> something |> #.find(selector)`.

<td>

```js
// Handle HTML strings
if (…)
  …
// Handle $(expr, $(...))
else if (!context || context.jquery)
  return (context || root).find(selector);
// Handle $(expr, context)
else
  return this.constructor(context)
    .find(selector);
```

From [jquery/src/core/init.js][]. The parallelism is much less clear here.

</table>

### Underscore.js

[Underscore.js][] is another utility library very widely used since 2009,
providing numerous functions that manipulate arrays, objects, and other
functions. It too has a codebase that transforms values through many expressions
– a codebase whose readability would therefore benefit from smart pipelines.

<table>
<thead>
<tr>
<th>With smart pipelines
<th>Status quo

<tbody>
<tr>
<td>

```js
function (obj, pred, context) {
  return obj
    |> isArrayLike(#)
    |> # ? _.findIndex : _.findKey
    |> #(obj, pred, context)
    |> (# !== void 0 && # !== -1)
        ? obj[#]
        : undefined;
}
```

<td>

```js
function (obj, pred, context) {
  var key;
  if (isArrayLike(obj)) {
    key = _.findIndex(obj, pred, context);
  } else {
    key = _.findKey(obj, pred, context);
  }
  if (key !== void 0 && key !== -1)
    return obj[key];
}
```

<tr>
<td>

```js
function (obj, pred, context) {
  return pred
  |> cb(#)
  |> _.negate(#)
  |> _.filter(obj, #, context);
}
```

<td>

```js
function (obj, pred, context) {
  return _.filter(obj,
    _.negate(cb(pred)),
    context
  );
}
```

<tr>
<td>

```js
function (
  srcFn, boundFn, ctxt, callingCtxt, args
) {
  if (!(callingCtxt instanceof boundFn))
    return srcFn.apply(ctxt, args);
  var self = srcFn
    |> #.prototype
    |> baseCreate(#);
  return self
  |> srcFn.apply(#, args)
  |> _.isObject(#) ? # : self;
}
```

<td>

```js
function (
  srcFn, boundFn,
  ctxt, callingCtxt, args
) {
  if (!(callingCtxt instanceof boundFn))
    return srcFn.apply(ctxt, args);
  var self = baseCreate(srcFn.prototype);
  var result = srcFn.apply(self, args);
  if (_.isObject(result)) return result;
  return self;
}
```

</table>

### Lodash

[Lodash][] is a fork of [Underscore.js][] that remains under rapid active
development. Along with Underscore.js’ other utility functions, Lodash provides
many other high-order functions that attempt to make [functional programming][]
more ergonomic. Like [jQuery][], Lodash is under the stewardship of the
[JS Foundation][], a member organization of TC39, through which Lodash’s developers
also have TC39 representation. And like jQuery and Underscore.js, Lodash’s API
involves complex data processing that becomes more readable with smart pipelines.

<table>
<thead>
<tr>
<th>With smart pipelines
<th>Status quo

<tbody>
<tr>
<td>

```js
function listCacheHas (key) {
  return this.__data__
    |> assocIndexOf(#, key)
    |> # > -1;
}
```

<td>

```js
function listCacheHas(key) {
  return assocIndexOf(this.__data__, key) > -1;
}
```

<tr>
<td>

```js
function mapCacheDelete (key) {
  const result = key
    |> getMapData(this, #)
    |> #['delete']
    |> #(key);
  this.size -= result ? 1 : 0;
  return result;
}
```

<td>

```js
function mapCacheDelete(key) {
  var result = getMapData(this, key)["delete"](key);
  this.size -= result ? 1 : 0;
  return result;
}
```

</table>

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
[formal pipeline specification]: https://xixixao.github.io/proposal-hack-pipelines
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
