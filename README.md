[![Build Status](https://secure.travis-ci.org/jneen/parsimmon.png)](http://travis-ci.org/jneen/parsimmon)

# Parsimmon

[![Parsimmon](http://i.imgur.com/wyKOf.png)](http://github.com/jneen/parsimmon)

**Authors:** [@jneen](https://github.com/jneen) and [@laughinghan](https://github.com/laughinghan)

**Maintainer:** [@wavebeem](https://github.com/wavebeem)

Parsimmon is a small library for writing big parsers made up of lots of little parsers. The API is inspired by parsec and Promises/A.

Parsimmon supports IE7 and newer browsers, along with Node.js. It can be used as a standard Node module through npm, or through `build/parsimmon.browser.min.js` directly in the browser through a script tag, where it exports a global variable called `Parsimmon` (you can also use the `build/parsimmon.browser.js` for local development).

## Quick Example

```javascript
var regex = Parsimmon.regex;
var string = Parsimmon.string;
var optWhitespace = Parsimmon.optWhitespace;
var lazy = Parsimmon.lazy;

function lexeme(p) {
  return p.skip(optWhitespace);
}

var lparen = lexeme(string('('));
var rparen = lexeme(string(')'));

var expr = lazy('an s-expression', function() { return form.or(atom); });

var number = lexeme(regex(/[0-9]+/).map(Number));
var id = lexeme(regex(/[a-zA-Z_]\w*/));

var atom = number.or(id);
var form = lparen.then(expr.many()).skip(rparen);

expr.parse('3').value;
// => 3

expr.parse('(add (mul 10 (add 3 4)) (add 7 8))').value;
// => ['add', ['mul', 10, ['add', 3, 4]], ['add', 7, 8]]
```

## Explanation

A Parsimmon parser is an object that represents an action on a stream of text, and the promise of either an object yielded by that action on success or a message in case of failure. For example, `string('foo')` yields the string `'foo'` if the beginning of the stream is `'foo'`, and otherwise fails.

The combinator method `.map` is used to transform the yielded value. For example,

```javascript
string('foo').map(function(x) { return x + 'bar'; })
```

will yield `'foobar'` if the stream starts with `'foo'`. The parser

```javascript
digits.map(function(x) { return parseInt(x) * 2; })
```

will yield the number 24 when it encounters the string '12'. The method `.result` can be used to set a constant result.

Calling `.parse(str)` on a parser parses the string, and returns an object with a `status` flag, indicating whether the parse succeeded. If it succeeded, the `value` attribute will contain the yielded value. Otherwise, the `index` and `expected` attributes will contain the index of the parse error (with `offset`, `line` and `column` properties), and a sorted, unique array of messages indicating what was expected.

The error object can be passed along with the original source to `Parsimmon.formatError(source, error)` to obtain a human-readable error string.

## Full API

### Included parsers / parser generators:
  - `Parsimmon.string("my-string")` is a parser that expects to find
    `"my-string"`, and will yield the same.
  - `Parsimmon.oneOf("abc")` is a parser that expects to find
    one of the characters `"a"`, `"b"`, or `"c"`, and will yield the same.
  - `Parsimmon.noneOf("abc")` is a parser that expects to find
    any character except one of the characters `"a"`, `"b"`, or `"c"`,
    and will yield the same.
  - `Parsimmon.regex(/myregex/, group=0)` is a parser that expects the stream
    to match the given regex, and yields the given match group, or the
    entire match.
  - `Parsimmon.succeed(result)` is a parser that doesn't consume any of
    the string, and yields `result`.
  - `Parsimmon.seq(p1, p2, ... pn)` accepts a variable number of parsers
    that it expects to find in order, yielding an array of the results.
  - `Parsimmon.seqMap(parser1, parser2, ..., function(result1, result2, ...) { return anotherResult; })`:
    matches all parsers sequentially, passing their results to the callback
    at the end, returning its value. Works like `seq` and `map` combined but
    without any arrays.
  - `Parsimmon.alt(p1, p2, ... pn)` accepts a variable number of parsers,
    and yields the value of the first one that succeeds, backtracking in between.
  - `Parsimmon.sepBy(content, separator)` accepts two parsers, and expects multiple `content`s, separated by `separator`s. Yields an array of `contents`.
  - `Parsimmon.sepBy1(content, separator)` same as `Parsimmon.sepBy`, but expects
    `content` to succeed at least once.
  - `Parsimmon.lazy(f)` accepts a function that returns a parser, which is
    evaluated the first time the parser is used. This is useful for
    referencing parsers that haven't yet been defined, and for implementing
    recursive parsers.
  - `Parsimmon.lazy(desc, f)` is the same as `Parsimmon.lazy` but also
    sets `desc` as the expected value (see `.desc()` below)
  - `Parsimmon.fail(message)` returns a failing parser with the given message.
  - `Parsimmon.letter` is equivalent to `Parsimmon.regex(/[a-z]/i)`
  - `Parsimmon.letters` is equivalent to `Parsimmon.regex(/[a-z]*/i)`
  - `Parsimmon.digit` is equivalent to `Parsimmon.regex(/[0-9]/)`
  - `Parsimmon.digits` is equivalent to `Parsimmon.regex(/[0-9]*/)`
  - `Parsimmon.whitespace` is equivalent to `Parsimmon.regex(/\s+/)`
  - `Parsimmon.optWhitespace` is equivalent to `Parsimmon.regex(/\s*/)`
  - `Parsimmon.any` consumes and yields the next character of the stream.
  - `Parsimmon.all` consumes and yields the entire remainder of the stream.
  - `Parsimmon.eof` expects the end of the stream.
  - `Parsimmon.index` is a parser that yields an object an object representing
    the current offset into the parse: it has a 0-based character `offset`
    property and 1-based `line` and `column` properties.
  - `Parsimmon.test(pred)` yield a single character if it passes the predicate.
  - `Parsimmon.takeWhile(pred)` yield a string containing all the next characters that pass the predicate.

### Adding base parsers

You can add a primitive parser (similar to the included ones) by using `Parsimmon.custom`. This is an example of how to create a parser that matches any character except the one provided:

```js
function notChar(char) {
  return Parsimmon.custom(function(success, failure) {
    return function(stream, i) {
      if (stream.charAt(i) !== char && i <= stream.length) {
        return success(i + 1, stream.charAt(i));
      }
      return failure(i, 'anything different than "' + char + '"');
    };
  });
}
```

This parser can then be used and composed the same way all the existing ones are used and composed, for example:

```js
var parser = Parsimmon.seq(Parsimmon.string('a'), notChar('b').times(5));
console.log(parser.parse('accccc'));
//=> {status: true, value: ['a', ['c', 'c', 'c', 'c', 'c']]}
```

### Parser methods
  - `parser.or(otherParser)`:
    returns a new parser which tries `parser`, and if it fails uses `otherParser`.
  - `parser.chain(function(result) { return anotherParser; })`:
    returns a new parser which tries `parser`, and on success calls the
    given function with the result of the parse, which is expected to
    return another parser, which will be tried next. This allows you
    to dynamically decide how to continue the parse, which is impossible
    with the other combinators.
  - `parser.then(anotherParser)`:
    expects `anotherParser` to follow `parser`, and yields the result
    of `anotherParser`. NB: the result of `parser` here is ignored.
  - `parser.map(function(result) { return anotherResult; })`:
    transforms the output of `parser` with the given function.
  - `parser.skip(otherParser)`
    expects `otherParser` after `parser`, but preserves the yield value
    of `parser`.
  - `parser.result(aResult)`:
    returns a new parser with the same behavior, but which yields `aResult`.
  - `parser.many()`:
    expects `parser` zero or more times, and yields an array of the results.
  - `parser.times(n)`:
    expects `parser` exactly `n` times, and yields an array of the results.
  - `parser.times(min, max)`:
    expects `parser` between `min` and `max` times, and yields an array
    of the results.
  - `parser.atMost(n)`:
    expects `parser` at most `n` times. Yields an array of the results.
  - `parser.atLeast(n)`:
    expects `parser` at least `n` times. Yields an array of the results.
  - `parser.mark()` yields an object with `start`, `value`, and `end` keys,
    where `value` is the original value yielded by the parser, and `start` and
    `end` are are objects with a 0-based `offset` and 1-based `line` and
    `column` properties that represent the position in the stream that
    contained the parsed text.
  - `parser.desc(description)` returns a new parser whose failure message is the passed
    description. For example, `string('x').desc('the letter x')` will indicate that
    'the letter x' was expected.

## Tips and patterns

These apply to most parsers for traditional languages - it's possible you may need to do something different for yours!

For most parsers, the following format is helpful:

1. define a `lexeme` function to skip all the stuff you don't care
   about (whitespace, comments, etc). You may need multiple types of lexemes.
   For example,

    ```javascript
    var ignore = whitespace.or(comment.many());
    function lexeme(p) { return p.skip(ignore); }
    ```

1. Define all your lexemes first. These should yield native javascript values.

    ```javascript
    var lparen = lexeme(string('('));
    var rparen = lexeme(string(')'));
    var number = lexeme(regex(/[0-9]+/)).map(parseInt);
    ```

1. Forward-declare one or more top-level expressions with `lazy`, referring to
   parsers that have not yet been defined. Generally, this takes the form of a
   large `.alt()` call

    ```javascript
    var expr = lazy('an expression', function() { return Parsimmon.alt(p1, p2, ...); });
    ```

    With `.lazy` you could also recursively refer to `expr` in its own
    definition.

1. Then build your parsers from the inside out - these should return
   AST nodes or other objects specific to your domain.

    ```javascript
    var p1 = ...
    var p2 = ...
    ```

1. Finally, export your top-level parser. Remember to skip ignored
   stuff at the beginning.

    ```javascript
    return ignore.then(expr.many());
    ```

### Fantasyland

[fantasyland]: https://github.com/fantasyland/fantasy-land "Fantasyland"
[fantasyland-logo]: https://github.com/fantasyland/fantasy-land/raw/master/logo.png

![][fantasyland-logo]

Parsimmon is also compatible with [fantasyland][]. It is a Semigroup, an Applicative Functor and a Monad.
