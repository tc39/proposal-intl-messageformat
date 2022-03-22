# Intl.MessageFormat

## Status

Champions: Eemeli Aro (Mozilla), Daniel Minor (Mozilla)

### Stage: 0

## Motivation

Intl.MessageFormat will build upon the
[MessageFormat 2.0](https://github.com/unicode-org/message-format-wg/) (hereafter MF2)
specification currently under development.
It will allow the use of MF2 resources to localize web sites,
enabling localization of the web using industry standard tooling and processes.
This should in turn make it easier to localize the web,
increasing the openness and accessibility of the web for speakers of languages
other than the handful of languages for which localization is typically done.

It is likely that eventually browsers themselves will be localized using MF2.
This is already planned for Firefox.
If this happens,
it will make sense to expose the MF2 implementation already present in the browser to the web,
rather than relying upon userland libraries.

## Use cases

- The primary use case is the retrieval of localized text ("a message")
  given a message identifier and a previously specified locale.

## API Description

The MF2 specification is still being developed by the working group.
The API below is based upon one proposal under consideration,
but should not be considered representative of a consensus among the working group.
In particular, the API shapes of
`MessageFormatOptions`, `MessageData`, `MessageResource`, and `ResolvedMessageFormatOptions`
will depend upon the data model chosen by the working group.

This proposal introduces one new primordial to ECMAScript, `Intl.MessageFormat`.
The other `interface` descriptions below are intended to represent plain objects.

### MessageData and MessageResource

The `MessageData` and `MessageResource` interfaces will be defined by
the MF2 data model developed by the MF2 working group.
`MessageData` contains a parsed representation of a single message for a particular locale.

A `MessageResource` is a group of related messages for a single locale.
Messages can be organized in a flat structure, or in hierarchy, using paths.
Conceptually, it is similar to a file containing a set of messages,
but there are no constrains implied on the underlying implementation.

```ts
interface MessageData {}

interface MessageResource {}
```

### MessageFormat

The `Intl.MessageFormat` constructor creates `MessageFormat` instances for a given locale,
`MessageFormatOptions` and a `MessageResource`.
The remaining operations are defined on `MessageFormat` instances.

If a string is used as the `resource` argument,
it will be parsed as a MF2 syntax representation of a message resource.

```ts
interface MessageFormat {
  new (
    resource: MessageResource | string,
    locales?: string | string[],
    options?: MessageFormatOptions
  ): MessageFormat;

  resolveMessage(
    msgPath: string | string[],
    values?: Record<string, unknown>,
    onError?: (error: Error, value: MessageValue) => void
  ): ResolvedMessage | undefined;

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
  resource: MessageResource;
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
returning a `ResolvedMessage` object or `undefined` if the message was not found.
This method has the following arguments:

- `msgPath` identifies the message from those available in the current resource.
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
  localeMatcher: "best fit" | "lookup" | undefined;
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
  type: "literal";
  value: string;
}

interface MessageNumber extends MessageValue {
  type: "number";
  value: number | bigint;
  options?: Intl.NumberFormatOptions & Intl.PluralRulesOptions;
  getPluralRule(): Intl.LDMLPluralRule;
  toParts(): Intl.NumberFormatPart[];
}

interface MessageDateTime extends MessageValue {
  type: "datetime";
  value: Date;
  options?: Intl.DateTimeFormatOptions;
  toParts(): Intl.DateTimeFormatPart[];
}

interface MessageFallback extends MessageValue {
  type: "fallback";
  value: undefined;
}

interface ResolvedMessage extends MessageValue {
  type: "message";
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
