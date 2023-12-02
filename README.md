# Intl.MessageFormat

## Status

Champions: Eemeli Aro (Mozilla/OpenJS Foundation), Daniel Minor (Mozilla)

### Stage: 1

#### Presentations

- 2022 March: [Stage 1 proposal](https://docs.google.com/presentation/d/1oThTeL_n5-HAfmJTri-i8yU2YtHUvj9AakmWiyRGPlw/edit?usp=sharing)
- 2023 October: [Stage 1 update](https://docs.google.com/presentation/d/15lwZipk0k5pMscSBbEPpMySsnM_qd4MOo_NqmmKyS-Q/edit?usp=sharing)

## Motivation

This proposal aims to make it easier to localize the web,
increasing the openness and accessibility of the web for speakers of all languages.
Currently, localization relies on a collection of mostly proprietary message formatting specifications
that are limited in their features and/or challenging for translators to work with.
Furthermore, localization often relies on parsing these custom formats
during the runtime or rendering of an application.

To help with this, we introduce
`Intl.MessageFormat` as a native parser and formatter for [MessageFormat 2.0] (aka “MF2”) messages.
MF2 is a specification currently being developed under the Unicode Consortium, with wide industry support.
This will allow for using MF2 messages to localize web sites,
enabling the localization of the web using industry standard tooling and processes.

In addition to a syntax that is designed to be accessible by both developers and translators,
MF2 defines a message data model that may be used to represent messages defined in any existing syntax.
This enables `Intl.MessageFormat` to be used within existing systems and workflows,
providing a shared message formatting runtime for all users.

[messageformat 2.0]: https://github.com/unicode-org/message-format-wg/

## Use cases

The primary use case is the retrieval and resolution of localized text (i.e. a “message”)
given a message source text, the locale and other options,
and optionally a set of runtime values.

Put together, this allows for any message ranging from the simplest to the most complex
to be defined by a developer, translated to any number of locales, and displayed to a user.

For instance, consider a relatively simple message such as

> You have 3 new notifications

In practice, this would need to account for any number of notifications,
and the plural rules of the current locale.
Using [MF2 syntax], this could be defined as:

[mf2 syntax]: https://github.com/unicode-org/message-format-wg/blob/develop/spec/syntax.md

```ini
.match {$count :number}
0   {{You have no new notifications}}
one {{You have {$count} new notification}}
*   {{You have {$count} new notifications}}
```

Some parts of the full message are explicitly repeated for each case,
as this makes it significantly easier for translators to work with the message.

In code, with the API proposed below, this would be used like this:

```js
const source = ... // string source of the message as above
const mf = new Intl.MessageFormat(source, 'en');
const notifications = mf.format({ count: 1 });
// 'You have 1 new notification'
```

As a majority of messages do not require multiple variants,
those are of course also supported by the proposed API:

```js
// A plain message
const mf1 = new Intl.MessageFormat('Hello!', 'en');
mf1.format(); // 'Hello!'

// A parametric message, formatted to parts
const mf2 = new Intl.MessageFormat('Hello {$place}!', 'en');
const greet = mf.formatToParts({ place: 'world' });
/* [
  { type: 'literal', value: 'Hello ' },
  { type: 'string', source: '$place', value: 'world' },
  { type: 'literal', value: '!' }
] */
```

More complex use cases and usage patterns are described within the API description.

## API Description

Though the MF2 specification is still being developed by the working group,
the API presented here is representative of its current consensus.
In particular, the exact shape of `MessageData` is still being discussed in the working group.

This proposal introduces one new primordial to ECMAScript, `Intl.MessageFormat`.
The other `interface` descriptions below are intended to represent plain objects.

### MessageData

The `MessageData` interface will be defined by
the MF2 data model developed by the MF2 working group.
It contains a parsed representation of a single message for a particular locale.

```ts
interface MessageData {}
```

### MessageFormat

The `Intl.MessageFormat` constructor creates a `MessageFormat` instance
for a `source` message, one or more locale identifiers,
and an optional `MessageFormatOptions` object.
If a string is used as the `source` argument,
it will be parsed as a MF2 syntax representation of a message.

Calling the constructor may throw an error if the `source` includes an MF2
[syntax or data model error](https://github.com/unicode-org/message-format-wg/blob/main/spec/formatting.md#error-handling).

```ts
interface MessageFormat {
  new (
    source: MessageData | string,
    locales?: string | string[],
    options?: MessageFormatOptions
  ): MessageFormat;

  format(
    values?: Record<string, unknown>,
    onError?: (error: Error) => void
  ): string;

  formatToParts(
    values?: Record<string, unknown>,
    onError?: (error: Error) => void
  ): MessagePart[];

  resolvedOptions(): ResolvedMessageFormatOptions;
}
```

#### Constructor options and resolvedOptions()

`MessageFormatOptions` contains configuration options
for the creation of `MessageFormat` instances.
The `ResolvedMessageFormatOptions` object contains the options
resolved during the construction of the `MessageFormat` instance.

Custom user-defined message formatting and selection functions may defined by the `functions` option.
These allow for any data types to be handled by custom functions.
Such functions may be referenced within messages,
and then called with the resolved values of their arguments and options.

```ts
interface MessageFormatOptions {
  functions?: { [key: string]: MessageFunction };
  localeMatcher?: 'best fit' | 'lookup';
}

interface ResolvedMessageFormatOptions {
  functions: { [key: string]: MessageFunction };
  locales: string[];
  localeMatcher: 'best fit' | 'lookup';
  message: MessageData;
}
```

#### format(values?, onError?)

As with other `Intl` formatters, `format()` returns a string.
This method has the following optional arguments:

- `values` provides variable values for the message's variable references.
- `onError` defines an error handler that will be called if
  message resolution or formatting fails.
  If `onError` is not defined,
  a warning will be issued for each error and a [fallback representation] used for the corresponding message part.

To determine the value `res` returned by the `format()` method,
the message is first resolved to a list of MessageValue instances.
Starting with an empty string `res`, for each MessageValue `mv`:

1. Let `strval` be the result of calling `mv.toString()`.
1. If the call fails or `strval` is not a string:
   1. Set `strval` to be the concatenation of `{`, `mv.source`, and `}`.
1. Append `strval` to the end of `res`.

#### formatToParts(values?, onError?)

For formatting a message to non-string targets,
the `formatToParts()` method is provided, returning an array of `MessagePart` objects.
This method has the following optional arguments:

- `values` provides variable values for the message's variable references.
- `onError` defines an error handler that will be called if
  message resolution or formatting fails.
  If `onError` is not defined,
  a warning will be issued for each error and a [fallback representation] used for the corresponding message part.

To determine the value `res` returned by the `formatToParts()` method,
the message is first resolved to a list of MessageValue instances.
Starting with an empty array `res`, for each MessageValue `mv`:

1. Let `parts` be the result of calling `mv.toParts()`.
1. If the call fails or `parts` is not an array:
   1. Set `parts` to be `[{ type: 'fallback', locale: 'und', source: mv.source }]`.
1. For each `part` or `parts`:
   1. Append `part` to `res`.

[fallback representation]: https://github.com/unicode-org/message-format-wg/blob/main/spec/formatting.md#fallback-resolution

### MessageValue

When formatting a message,
the selectors and placeholders of a message are each resolved first
to an intermediate `MessageValue` representation.
This can be thought of as an immutable object with properties and methods,
though its JavaScript representation is only available to custom functions.

```ts
interface MessageValue {
  type: string;
  locale: string;
  source: string;
  options?: { [key: string]: unknown };
  selectKeys?: (keys: string[]) => string[];
  toParts?: () => MessagePart[];
  toString?: () => string;
  valueOf?: () => unknown;
}

type MessagePart =
  | { type: 'text'; value: string }
  | ({
      type: string;
      source: string;
      locale?: string;
    } & (
      | { value?: unknown }
      | { parts: Array<{ type: string; value: unknown; source?: string }> }
    ));
```

A `MessageValue` is an object with a string `type`, a string `locale` identifier,
and an opaque `source` string identifying its origin.
All other fields are optional;
they determine how the value may be used in MF2 expressions.

In order to be usable as a formatted placeholder,
the object (or its prototype chain) MUST include a `toString` method returning a string
and a `toParts` method returning an array of `MessagePart`s.
All of the built-in implementations of this method return an array with exactly one value,
but user-defined functions may return any number of parts, or none.

Except for parts corresponding to literal values,
each `MessagePart` MUST include the `type` and `source` from the `MessageValue`.
It MAY also include a string `locale` identifier,
and optionally either an explicit `value` of any type
or its own sequence of `parts`.

In order to be usable as a variant selector,
the `MessageValue` object MUST include a `selectKeys` method.
When called with an array of string keys,
it MUST return an array whose elements are a subset of those keys.
The returned keys will be considered to match the selector and be in preferential order.

Within a message, the value of an expression may be assigned to a message-local variable,
and such a variable may be used as an input argument to another expression,
or as an option value.
Some functions (including the default `number`) accept
objects with a `valueOf` method and an `options` field
in addition to `type` and `locale` as input.
These MAY also be defined on the returned object.

#### Literal Text

Text in patterns outside expressions is always literal.
While its resolved value is never presented as JS,
for the sake of simplicity it may be thought of as having the following resolved value:

```ts
interface MessageText {
  type: 'text';
  locale: string;
  source: string;
  toParts(): [MessageTextPart];
  toString(): string;
}

interface MessageTextPart {
  type: 'text';
  value: string;
}
```

For `MessageText`, the value returned by `toString()` and
the `value` field of the object returned by `toParts()`
corresponding to the text source.
Its `locale` is always the same as the message's base locale.

#### Expressions

Expressions are used as selectors and as pattern placeholders.
A local variable declaration may assign the value of an expression to a local variable,
allowing for the same expression to be used in multiple places
and potentially with different roles.

An expression may have one of three forms:

- An operand (either a literal value or a variable reference).
- An operand with an annotation.
- An annotation with no operand.

The resolution of annotations using the `:` prefix is customisable
using the constructor's `functions` option,
which takes `MessageFunction` function values that are applied when
the annotation's name (without the `:`) corresponds to the `functions` key.

### MessageFunction

Fundamentally, messages are formed by concatenating values together.
In order to support the handling of user-defined value types with user-defined formatting options,
as well as other needs,
user-provided message functions may be provided via the constructor's `functions` option
to complement or replace the default ones.

```ts
type MessageFunction = (
  source: string,
  locales: string[],
  options: { [key: string]: unknown },
  input?: unknown
) => MessageValue;
```

The `input` and `options` values are constructed as follows:

- If the value is a literal defined in the message syntax,
  the value is its `string` value.
- If the value is a variable referring to a local variable declaration,
  the value is the `MessageValue` that the declaration's expression resolves to.
- Otherwise, the value is a variable referring to an external value,
  and its type and value are that of the external value.

As function options are often set by literal values,
and as MF2 considers all literals to be strings,
the JSON string representation should be supported for numerical and boolean values
when used as an input or as an option value.
Each function will need to parse these separately from their string representations.

If a function is locale-dependent,
it should accept an extra option key called `locale`
which resolves to a string overriding the message's base locale.
This option value should always be parsed as an array or
(if a single string) a comma-delimited list of BCP 47 locale identifiers.

#### Default Functions

Two commonly used message functions `number` and `string` are provided as a starting point,
and as handlers for placeholders without an annotation,
such as variable references like `{$foo}` or literal values like `{|the bar|}`.

Variable references are resolved by first looking for a local variable declaration
matching its name, then by looking in the `values` argument.
Literal values always resolve to strings.

As selector expressions must have an annotation or contain a variable reference
that references a variable declaration within the same message with an annotation,
un-annotated expressions only need to be considered for formattable placeholders.

If a placeholder expression contains a variable reference without an annotation
and the variable resolves to a number or bigint value or a Number instance,
it resolves instead to the result of calling the `number` function
with the numerical value as input and no options.

If a placeholder expression would resolve to a string value or a String instance,
it resolves instead to the result of calling the `string` function
with the value as input and no options.

Otherwise, un-annotated values resolve to the following shape:

```ts
interface MessageUnknownValue {
  type: 'unknown';
  locale: string;
  source: string;
  toParts(): [MessageUnknownPart];
  toString(): string;
  valueOf(): unknown;
}

interface MessageUnknownPart {
  type: 'unknown';
  source: string;
  value: unknown;
}
```

With `MessageUnknownValue`,
the `toString()` method uses the equivalent of `String()` to format the value
while `valueOf()` returns the original value.

If the constructor options of a MessageFormat instance include a `functions` value
that overrides the default `number` or `string` functions,
those will be called instead of the default ones, as appropriate.

#### `number`

Accepts as input any of the following:

- A number or a bigint.
  This is used directly as the `value`.
- An object with a `valueOf()` method that returns a number or a bigint,
  which is then used as the `value`.
- The JSON string representation of a number, to accommodate literal values as in `{42 :number}`.
  The `value` is determined by the equivalent of calling `JSON.parse()` on the string,
  and asserting that it returns a number or a bigint.

Returns a fallback value if invoked without such an input,
or if an error occurs while determining `value`.

Internally, constructs a `locales` array of strings as used by
the Intl.NumberFormat and Intl.PluralRules constructors.
This will include, in order:

1. Any locales set by the expression's `"locale"` option.
2. The locale of the input, if it is an object and has a string or string array property `"locale"`.
3. The base locale or locale chain of the message.

To determine the formatting and selection options for the number,
a new empty `options` object is created.
If the input is an object with a property `"options"` with an object value,
the `options` object is extended with its values.
Then, a primitive value is set for each of the expression's options except `"locale"`:

- For each option value, if it is an object, coerce it to a string.
- For `useGrouping`, convert `"true"` and `"false"` to their corresponding boolean values.
- For `roundingIncrement`, `minimumIntegerDigits`, and `(minimum|maximum)(Fraction|Significant)Digits`,
  parse a string value as a non-negative integer number.

Returns a value with the following shape:

```ts
interface MessageNumber {
  type: 'number';
  locale: string;
  source: string;
  options: Intl.NumberFormatOptions & Intl.PluralRulesOptions;
  selectKeys(keys: string[]): string[];
  toParts(): [MessageNumberPart];
  toString(): string;
  valueOf(): number | bigint;
}

interface MessageNumberPart {
  type: 'number';
  locale?: string;
  source: string;
  parts: Intl.NumberFormatPart[];
}
```

When a `MessageNumber` is used as a selector (calling its `selectKeys()` method),
a key with an exact numeric match to the value will be preferred over a key matching the value's plural category
(some of `zero`, `one`, `two`, `few`, `many`, and `other`, depending on the locale).

When a `MessageNumber` is being formatted,
calling its `toString()` will return a string corresponding to calling

```js
new Intl.NumberFormat(locales, options).format(value);
```

and calling its `toParts()` method will return an array with a single object member
where the `parts` will correspond to the results of calling

```js
new Intl.NumberFormat(locales, options).formatToParts(value);
```

#### `string`

Accepts any input, and parses any non-string value using `String()`.
For no input, resolves its value to an empty string.
On error, resolves to a fallback value.

Accepts only the `locale` option as an override for the message's locale.

Returns a value with the following shape:

```ts
interface MessageString {
  type: 'string';
  locale: string;
  source: string;
  selectKeys(keys: string[]): [] | [string];
  toParts(): [MessageStringPart];
  toString(): string;
  valueOf(): string;
}

interface MessageStringPart {
  type: 'string';
  locale?: string;
  source: string;
  value: string;
}
```

When a `MessageString` is used as a selector,
the returned array may include at most one entry,
if one of the keys was an exact string match for the value.

### Fallback Values

It's possible for a `MessageFunction` call to throw an error,
or for a `format()` or `formatToParts()` call to throw an error.
When this happens,
the error is caught and a user-provided `onError` handler is called.
If no such handler is defined,
the default behaviour is to issue a warning rather than throwing the error.
This allows for some formatted representation to always be provided.

In such a case, a fallback representation is used instead for the value:

```ts
interface MessageFallback {
  type: 'fallback';
  locale: 'und';
  source: string;
  toParts(): [MessageFallbackPart];
  toString(): string;
}

interface MessageFallbackPart {
  type: 'fallback';
  source: string;
}
```

This representation is also used when resolving MF2 expressions
that include "markup", "reserved" or "private-use" annotations.

The `source` of the `MessageFallback` corresponds to the `source` of the `MessageValue`.
When `MessageFallback` is formatted to a string,
its value is the concatenation of a left curly brace `{`, the `source` value,
and a right curly brace `}`.

## Comparison

The MF2 specification is being developed based upon lessons learned from existing systems
including [ICU MessageFormat] and [Fluent].

The implementation of Fluent within Firefox mostly relies upon
a declarative syntax in the DOM,
but it does provide an [API] for retrieving messages directly from Fluent
when not being used to localize the DOM.

[icu messageformat]: https://unicode-org.github.io/icu/userguide/format_parse/messages/
[fluent]: https://projectfluent.org/
[api]: https://firefox-source-docs.mozilla.org/l10n/fluent/tutorial.html#non-markup-localization

## Implementations

### Polyfill/transpiler implementations

The MessageFormat 2.0 specification is under development.
A polyfill implementation of this proposal is available under the
[messageformat](https://github.com/messageformat/messageformat/tree/master/packages/mf2-messageformat) project.
