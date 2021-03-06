# markdown-it design principles

## Data flow

Input data is parsed via nested chains of rules. There are 3 nested chains -
`core`, `block` & `inline`:

```
core
    core.rule1 (normalize)
    ...
    core.ruleX

    block
        block.rule1 (blockquote)
        ...
        block.ruleX

    core.ruleX1 (intermediate rule that applies on block tokens, nothing yet)
    ... 
    core.ruleXX

    inline (applied to each block token with "inline" type)
        inline.rule1 (text)
        ...
        inline.ruleX

    core.ruleYY (applies to all tokens)
    ... (abbreviation, footnote, typographer, linkifier)

```

The result of the parsing is a *list of tokens*, that will be passed to the `renderer` to generate the html content.

These tokens can be themselves parsed again to generate more tokens (ex: a `list token` can be divided into multiple `inline tokens`).

An `env` sandbox can be used alongside tokens to inject external variables for your parsers and renderers.

Each chain (core / block / inline) uses an independent `state` object when parsing data, so that each parsing operation is independent and can be disabled on the fly.


## Token stream

Instead of traditional AST we use more low-level data representation - *tokens*.
The difference is simple:

- Tokens are a simple sequence (Array).
- Opening and closing tags are separate tokens.
- There are special token objects, "inline containers", having nested tokens.
  sequences with inline markup (bold, italic, text, ...).

Each token has some common fields:

- __type__ - token name.
- __level__ - nesting level, useful to seek closing pair.
- __lines__ - [begin, end], for block tokens only. Range of input lines,
  compiled to this token.

Inline tokens (`type === "inline"`) have additional properties:

- __content__ - raw text, unparsed inline content.
- __children__ - nested token stream.

In total, a token stream is:

- On the top level - array of paired or single "block" tokens:
  - open/close for headers, lists, blockquotes, paragraphs, ...
  - codes, fenced blocks, horizontal rules, html blocks, inlines containers
- Each inline token have a `.children` property with a nested token stream for inline content:
  - open/close for strong, em, link, code, ...
  - text, line breaks

Why not AST? Because it's not needed for our tasks. We follow KISS principle.
If you wish - you can call a parser without a renderer and convert the token stream
to an AST.

More details about tokens:

- [Renderer source](https://github.com/markdown-it/markdown-it/blob/master/lib/renderer.js)
- [Live demo](https://markdown-it.github.io/) - type your text ant click `debug` tab.


## Rules

Rules are functions, doing "magic" with parser `state` objects. A rule is associated with one or more *chains* and is unique. For instance, a `blockquote` token is associated with `blockquote`, `paragraph`, `heading` and `list` chains.

Rules are managed by names via [Ruler](https://markdown-it.github.io/markdown-it/#Ruler) instances and can be  `enabled` / `disabled` from the [MarkdownIt](https://markdown-it.github.io/markdown-it/#MarkdownIt) methods.

You can note, that some rules have a `validation mode` - in this mode rules do not
modify the token stream, and only look ahead for the end of a token. It's one
important design principle - a token stream is "write only" on block & inline parse stages.

Parsers are designed to keep rules independent of each other. You can safely enable/disable them, or
add new ones. There are no universal recipes for how to create new rules - design of
distributed state machines with good data isolation is a tricky business. But you
can investigate existing rules & plugins to see possible approaches.

Also, in complex cases you can try to ask for help in tracker. Condition is very
simple - it should be clear from your ticket, that you studied docs, sources,
and tried to do something yourself. We never reject with help to real developpers.


## Renderer

After token stream is generated, it's passed to a [renderer](https://github.com/markdown-it/markdown-it/blob/master/lib/renderer.js).
It then plays all the tokens, passing each to a rule with the same name as token type.

Renderer rules are located in `md.renderer.rules[name]` and are simple functions
with the same signature:

```js
function (tokens, idx, options, env, renderer) {
  //...
  return htmlResult;
}
```

In many cases that allows easy output change even without parser intrusion.
For example, let's replace images with vimeo links to player's iframe:

```js
var md = require('markdown-it')();

var defaultRender = md.renderer.rules.image,
    vimeoRE       = /^https?:\/\/(www\.)?vimeo.com\/(\d+)($|\/)/;

md.renderer.rules.image = function (tokens, idx, options, env, self) {
  var id;

  if (vimeoRE.test(tokens[idx].href)) {

    id = tokens[idx].href.match(vimeoRE)[2];

    return '<div class="embed-responsive embed-responsive-16by9">\n' +
           '  <iframe class="embed-responsive-item" src="//player.vimeo.com/video/' + id + '"></iframe>\n' +
           '</div>\n';
  }

  return defaultRender(tokens, idx, options, env, self);
});
```

You also can write your own renderer to generate other formats than HTML, such as JSON/XML... You can even use it to generate AST.


## Summary

This was mentioned in [Data flow](#data-flow), but let's repeat sequence again:

1. Blocks are parsed, and top level of token stream filled with block tokens.
2. Content on inline containers is parsed, filling `.children` properties.
3. Rendering happens.

And somewhere between you can apply additional transformations :) . Full content
of each chain can be seen on the top of
[parser_core.js](https://github.com/markdown-it/markdown-it/blob/master/lib/parser_core.js),
[parser_block.js](https://github.com/markdown-it/markdown-it/blob/master/lib/parser_block.js) and
[parser_inline.js](https://github.com/markdown-it/markdown-it/blob/master/lib/parser_inline.js)
files.

Also you can change output directly in [renderer](https://github.com/markdown-it/markdown-it/blob/master/lib/renderer.js) for many simple cases.
