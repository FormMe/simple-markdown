simple-markdown
===============

Javascript markdown parsing, made simple.


Philosophy
----------

Most markdown-like parsers aim for [speed][marked] or
[edge case handling][CommonMark].
simple-markdown aims for extensibility and simplicity.

[marked]: https://github.com/chjj/marked
[CommonMark]: https://github.com/jgm/CommonMark

What does this mean?
Many websites using markdown-like languages have custom extensions,
such as `@`mentions or issue number linking. These extensions are
usually custom to the specific site, and don't make sense as part
of generic markdown. Unfortunately, most markdown-like parsers
have a very small interface that doesn't allow extension without
forking. simple-markdown is designed to allow simple addition of
custom extensions without needing to be forked.

At Khan Academy, we use markdown to format
over half of our math exercises, but we need extensions for
math text and interactive widgets.


Basic parsing/output API
------------------------

#### `SimpleMarkdown.defaultBlockParse(source)`

Returns a syntax tree of the result of parsing `source` with the
default markdown rules. Assumes a block scope.

#### `SimpleMarkdown.defaultInlineParse(source)`

Returns a syntax tree of the result of parsing `source` with the
default markdown rules, where `source` is assumed to be inline text.
Does not emit `<p>` elements. Useful for allowing inline markdown
formatting in one-line fields where paragraphs, lists, etc. are
disallowed.

#### `SimpleMarkdown.defaultImplicitParse(source)`

Parses `source` as block if it ends with `\n\n`, or inline if not.

#### `SimpleMarkdown.defaultOutput(syntaxTree)`

Returns React-renderable output for `syntaxTree`.

*Note: raw html output will be coming soon*


Extension Overview
------------------

Elements in simple-markdown are generally created from rules.
For parsing, rules must specify `match` and `parse` methods.
For output, rules must specify an `output` method.

Here is an example rule, a slightly modified version of what
simple-markdown uses for parsing **strong** (**bold**) text:

    strong: {
        match: function(source, state, lookbehind) {
            return /^\*\*([\s\S]+?)\*\*(?!\*)/.exec(source);
        },
        parse: function(capture, recurseParse, state) {
            return {
                content: recurseParse(capture[1], state)
            };
        },
        output: function(node, recurseOutput) {
            return React.DOM.strong(null, recurseOutput(node.content));
        }
    },

Let's look at those three methods in more detail.

#### `match(source, state, lookbehind)`

simple-markdown calls your `match` function to determine whether the
upcoming markdown source matches this rule or not.

`source` is the upcoming source, beginning at the current position of
parsing (source[0] is the next character).

`state` is a mutable state object to allow for more complicated matching
and parsing. The most common field on `state` is `inline`, which all of
the default rules set to true when we are in an inline scope, and false
or undefined when we are in a block scope.

`lookbehind` is the string previously captured at this parsing level, to
allow for lookbehind. For example, lists check that lookbehind ends with
`/^$|\n *$/` to ensure that lists only match at the beginning of a new
line.

If this rule matches, `match` should return an object, array, or
array-like object, which we'll call `capture`, where `capture[0]`
is the full matched source, and any other fields can be used in the
rule's `parse` function. The return value from `Regexp.prototype.exec`
fits this requirement, and the common use case is to return the result
of `someRegex.exec(source)`.

If this rule does not match, `match` should return null.

NOTE: If you are using regexes in your match function, your regex
should always begin with `^`. Regexes without leading `^`s can
cause unexpected output or infinite loops.

#### `parse(capture, recurseParse, state)`

`parse` takes the output of `match` and transforms it into a syntax
tree node object, which we'll call `node` here.

`capture` is the non-null result returned from match.

`recurseParse` is a function that can be called on sub-content and
state to recursively parse the sub-content. This returns an array.

`state` is the mutable state threading object, which can be examined
or modified, and should be passed through to any `recurseParse` calls.

For example, to parse inline sub-content, you can add `inline: true`
to state, or `inline: false` to force block parsing (to leave the
parsing scope alone, you can just pass `state` with no modifications).
For example:

    var innerText = capture[1];
    recurseParse(innerText, _.defaults({
        inline: true
    }, state));
    
`parse` should return a `node` object, which can have custom fields
that will be passed to `output`, below. The one reserved field is
`type`, which designates the type of the node, which will be used
for output. If no type is specified, simple-markdown will use the
current rule's type (the common case). If you have multiple ways
to parse a single element, it can be useful to have multiple rules
that all return nodes of the same type.

#### `output(node, recurseOutput)`

`output` takes a syntax tree `node` and transforms it into
React-renderable output.

`node` is the return value from `parse`, which has a type
field of the same type as the current rule, as well as any
custom fields created by `parse`.

`recurseOutput` is a function to recursively output sub-tree
nodes created by using `recurseParse` in `parse`.

The simple-markdown API contains several helper methods for
creating rules, as well as methods for creating parsers and
outputters from rules.

Extension API
-------------

simple-markdown includes access to the default list of rules,
as well as several functions to allow you to create parsers and
outputters from modifications of those default rules, or even
from a totally custom rule list.

These functions are separated so that you can customize
intermediate steps in the parsing/output process, if necessary.

#### `SimpleMarkdown.defaultRules`

The default rules, specified as an object, where the keys are
the rule types, and the values are objects containing `match`,
`parse`, and `output` functions (these rules can be used for
both parsing and outputting).

#### `SimpleMarkdown.defaultPriorities`

An array of the types in `defaultRules`, in priority order.
To extend simple-markdown with your own rules, you will need
to create a `priorities` array of the order in which to parse
rules (rules will only be considered if they are found in the
`priorities` array). If you intend to keep all or most of the
default rules, creating a custom `priorities` array from the
`defaultPriorities` array is often useful.

The `priorities` array is an array of strings, where each element
is the string name of a type in the `rules` object.

#### `SimpleMarkdown.parserFor(rules, priorities)`

Takes a `rules` object and a `priorities` array, and
returns a parser for the rule types in the `priorities`
array, in that order. The rules in the `rules` object
should contain `match` and `parse` functions for each
rule listed in the `priorities` array.

#### `SimpleMarkdown.ruleOutput(rules)`

Takes a `rules` object and returns a function that can
output a single syntax tree node of any type that is
in the `rules` object, given a node and a recursive output
function. This is not the final output function because it
doesn't handle arrays of nodes or recursion
(see `outputFor`).

#### `SimpleMarkdown.outputFor(singleNodeOutputFunction)`

Takes a function that can output any single syntax tree
node and returns a function that maps over syntax tree
arrays correctly.

The most common use case is to pass the output of
`ruleOutput` as the parameter to `outputFor`:

    var output = SimpleMarkdown.outputFor(SimpleMarkdown.ruleOutput(rules));

#### Putting it all together

Given a set of rules and priorities, one can create a single
function that takes an input content string and outputs a
React-renderable as follows. Note that since many rules
expect blocks to end in `"\n\n"`, we append that to source
input manually, in addition to specifying `inline: false`
(`inline: false` is technically optional for all of the
default rules, which assume `inline` is false if it is
undefined).

    var rules = SimpleMarkdown.defaultRules; // for example
    var priorities = SimpleMarkdown.defaultPriorities; // likewise
    var parser = SimpleMarkdown.parserFor(rules, priorities);
    var output = SimpleMarkdown.outputFor(SimpleMarkdown.ruleOutput(rules));
    var blockParseAndOutput = function(source) {
        // Many rules require content to end in \n\n to be interpreted
        // as a block.
        var blockSource = source + "\n\n";
        var parseTree = parser(blockSource, {inline: false});
        var outputResult = output(parseTree);
        return outputResult;
    };

Extension rules helper functions
--------------------------------

*Coming soon*

LICENSE
-------
MIT. See the LICENSE file for text.
