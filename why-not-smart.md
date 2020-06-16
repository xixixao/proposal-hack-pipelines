# Why this proposal is NOT so "Smart"

Or why it avoids the "bare style".

Here's a TLDR.

This is the intro in the README of [proposal-smart-pipelines](https://github.com/js-choi/proposal-smart-pipelines) to the "smart" style:

> This proposal introduces a new binary pipe operator |> to JavaScript...

```js
value
|> await #
|> doubleSay(#, ', ')
|> capitalize // This is a function call.
|> # + '!'
|> new User.Message(#)
|> await #
|> console.log; // This is a method call.
```

And this is the intro to this proposal:

> This proposal introduces a new binary pipe operator |> to JavaScript...

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

Notice the lack of comments.

If we need to add a comment to explain a subset of the syntax, it's a clear signal this syntax is not obvious enough.

Because the syntax adds so little, this proposal removes it altogether. See more reasons in this [comment](https://github.com/tc39/proposal-pipeline-operator/issues/167#issuecomment-631878219).
