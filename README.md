# Intl.MessageFormat

## Status

Champions: Eemeli Aro (Mozilla/OpenJS Foundation), Daniel Minor (Mozilla)

### Stage: 1

## Motivation

This proposal aims to make it easier to localize the web,
increasing the openness and accessibility of the web for speakers of all languages.
Currently, localization relies on a collection of mostly proprietary message formatting specifications
that are limited in their features and/or challenging for translators to work with.
Furthermore, localization often relies on parsing these custom formats
during the runtime or rendering of an application.

To help with this, we introduce
`Intl.MessageFormat` as a native parser and formatter for [MessageFormat 2.0] (aka "MF2") messages.
MF2 is a specification currently being developed under the Unicode Consortium, with wide industry support.
This will allow for using MF2 messages to localize web sites,
enabling the localization of the web using industry standard tooling and processes.

In addition to a syntax that is designed to be accessible by both developers and translators,
MF2 defines a message data model that may be used to represent messages defined in any existing syntax.
This enables `Intl.MessageFormat` to be used within existing systems and workflows,
providing a shared message formatting runtime for all users.

[messageformat 2.0]: https://github.com/unicode-org/message-format-wg/

## Use cases

The primary use case is the retrieval and resolution of localized text (i.e. a "message")
given a message source text, the locale and other options, a message identifier,
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
# Note! MF2 syntax is still under development; this may still change

match {$count}
when 0   {You have no new notifications}
when one {You have one new notification}
when *   {You have {$count} new notifications}
```

Some parts of the full message are explicitly repeated for each case,
as this makes it significantly easier for translators to work with the message.

In code, with the API proposed below, this would be used like this:

```js
const source = ... // string source of the message as above
const mf = new Intl.MessageFormat(source, ['en']);
const notifications = mf.resolveMessage({ count: 1 });
notifications.toString(); // 'You have one new notification'
```

As a majority of messages do not require multiple variants,
those are of course also supported by the proposed API:

```js
// A plain message
const mf1 = new Intl.MessageFormat('{Hello!}', ['en']);
const greet = mf.resolveMessage();
greet.toString(); // 'Hello!'

// A parametric message
const mf2 = new Intl.MessageFormat('{Hello {$place}!}', ['en']);
const greet = mf.resolveMessage({ place: 'world' });
greet.toString(); // 'Hello world!'
```

More complex use cases and usage patterns are described within the API description.

## API Description

The MF2 specification is still being developed by the working group.
The API below is based upon one proposal under consideration,
but should not be considered representative of a consensus among the working group.
In particular, the API shapes of
`MessageFormatOptions`, `MessageData`, and `ResolvedMessageFormatOptions`
will depend upon the data model chosen by the working group.

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

The `Intl.MessageFormat` constructor creates `MessageFormat` instances for a given locale,
`MessageFormatOptions` and a `MessageData` structure.
If a string is used as the `source` argument,
it will be parsed as a MF2 syntax representation of a message.

```ts
interface MessageFormat {
  new (
    source: MessageData | string,
    locales?: string | string[],
    options?: MessageFormatOptions
  ): MessageFormat;

  resolveMessage(
    values?: Record<string, unknown>,
    onError?: (error: Error, value: MessageValue) => void
  ): ResolvedMessage;

  resolvedOptions(): ResolvedMessageFormatOptions;
}
```

#### Constructor options and resolvedOptions()

The interfaces for
`MessageFormatOptions` and `ResolvedMessageFormatOptions`
will depend on the final MF2 data model.
`MessageFormatOptions` contains configuration options
for the creation of `MessageFormat` instances.
The `ResolvedMessageFormatOptions` object contains the options
resolved during the construction of the `MessageFormat` instance.

Custom user-defined message formatting function may defined by the `formatters` option.
These allow for any data types to be handled by custom functions.
Formatting functions may be referenced within messages,
and then called with the resolved values of their arguments and options.
At least two formatting functions are to be provided by the implementation:

- `number`, returning a `MessageNumber`
- `datetime`, returning a `MessageDateTime`

These may be shadowed by user-defined functions defined in the formatters option.

```ts
interface MessageFormatOptions {
  formatters?: Record<string, MessageFormatterFunction>;
  localeMatcher?: 'best fit' | 'lookup';
  ...
}

interface ResolvedMessageFormatOptions {
  locales: string[],
  localeMatcher: 'best fit' | 'lookup';
  message: MessageData;
  ...
}

type MessageFormatterFunction = (
  locales: string[],
  options: Record<string, unknown>,
  ...args: MessageValue[]
) => MessageValue
```

#### resolveMessage()

For formatting a message, the `resolveMessage()` method is provided,
returning a `ResolvedMessage` object.
This method has the following arguments:

- `values` are to lookup variable references used in the `MessageData`.
- `onError` argument defines an error handler that will be called if
  message resolution or formatting fails.
  If `onError` is not defined,
  errors will be ignored and a fallback representation used for the corresponding message part.

### MessageValue

`ResolvedMessage` is intended to provide a building block for the localization of messages anywhere,
including contexts where its representation as a plain string would not be sufficient.
`ResolvedMessage` extends a base interface `MessageValue`,
with its `value` property providing an iterator of other `MessageValue`s.

The `source` of a `MessageValue` provides an opaque identifier for the value,
such as `"$foo"` for a variable.
The `meta` of a `MessageValue` is a map of resolved metadata for the value in question.
Each `MessageValue` provides a `toString()` method;
for numerical and date values a `toParts()` method is also provided,
taking into account the locale context as well as any formatting options.
`MessageFallback` is used when the resolution of a part of the message failed.

Values matching the structure of `MessageNumber` and `MessageDateTime`
(i.e. with the corresponding `type`, `value` and `options` fields)
may be used in the `values` argument as partially formatted values.

```ts
interface LocaleContext {
  locales: string[];
  localeMatcher: 'best fit' | 'lookup' | undefined;
}

interface MessageValue {
  type: string;
  value: unknown;
  localeContext?: LocaleContext;
  source?: string;
  meta?: Record<string, string>;
  toString(): string;
}

interface MessageLiteral extends MessageValue {
  type: 'literal';
  value: string;
}

interface MessageNumber extends MessageValue {
  type: 'number';
  value: number | bigint;
  options?: Intl.NumberFormatOptions & Intl.PluralRulesOptions;
  getPluralRule(): Intl.LDMLPluralRule;
  toParts(): Intl.NumberFormatPart[];
}

interface MessageDateTime extends MessageValue {
  type: 'datetime';
  value: Date;
  options?: Intl.DateTimeFormatOptions;
  toParts(): Intl.DateTimeFormatPart[];
}

interface MessageFallback extends MessageValue {
  type: 'fallback';
  value: undefined;
}

interface ResolvedMessage extends MessageValue {
  type: 'message';
  value: Iterable<MessageValue>;
}
```

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
An experimental implementation of one proposal for the MessageFormat 2.0 specification is available under the
[messageformat](https://github.com/messageformat/messageformat/tree/master/packages/mf2-messageformat) project.
