# Intl.MessageFormat

## Status

Champions: Eemeli Aro (Mozilla), Daniel Minor (Mozilla)

### Stage: 0

## Motivation

Intl.MessageFormat will build upon the [MessageFormat 2.0](https://github.com/unicode-org/message-format-wg/) (hereafter MF2)
specification currently under development. It will allow the use of MF2 resources
to localize web sites, enabling localization of the web using industry standard tooling and
processes. This should in turn make it easier to localize the web, increasing the openness
and accessibility of the web for speakers of languages other than the handful of languages for
which localization is typically done.

It is likely that eventually browsers themselves will be localized using MF2. This is already
planned for Firefox. If this happens, it will make sense to expose the MF2 implementation
already present in the browser to the web, rather than relying upon userland libraries.

## Use cases

- The primary use case is the retrieval of localized text ("a message") given a message
  identifier and a previously specified locale.

## Description

### API

The MF2 specification is still being developed by the working group. The API below is based
upon one proposal under consideration, but should not be considered representative of a
consensus among the working group. In particular, the API shape of `MessageFormatOptions`,
`Message`, `Scope` and `ResolvedOptions` will depend upon the data model chosen by the
working group.

The interface provided by `Message` will be defined by the MF2 data model developed by
the MF2 working group. It contains localized text for a particular locale.

```ts
interface Message {}
```

A `Resource` is a group of related messages for a single locale. Messages can be organized
in a flat structure, or in hierarchy, using paths. Conceptually, it is similar to a file
containing a set of messages, but there are no constrains implied on the underlying
implementation.

```ts
interface Resource {
  getId(): string;

  getMessage(path: string[]): Message | undefined;
}
```

The `Intl.MessageFormat` constructor creates `MessageFormat` instances for a given locale,
`MessageFormatOptions` and an optional set of `Resource`s. The remaining operations are
defined on `MessageFormat` instances.

The interfaces for `MessageFormatOptions`, `Scope` and `ResolvedOptions` will be defined
by the MF2 data model. `MessageFormatOptions` contains configuration options for the
creation of `MessageFormat` instances. The `Scope` object is used to lookup variable
references used in the `Message`. The `ResolvedOptions` object contains the options
resolved during the construction of the `MessageFormat` instance.

```ts
interface MessageFormatOptions { }

interface Scope { }

interface Intl.MessageFormat {
  new (
    locales: string | string[],
    options?: MessageFormatOptions | null,
    ...resources: Resource[]
  ): Intl.MessageFormat;

  addResources(...resources: Resource[]);

  format(msgPath: string | string[], scope?: Scope): string;
  format(resId: string, msgPath: string | string[], scope?: Scope): string;

  formatToParts(
    resId: string,
    msgPath: string | string[],
    scope?: Scope
  ): MessageFormatPart[];

  resolvedOptions(): ResolvedOptions;
}
```

## Comparison

The MF2 specification is being developed based upon lessons learned from existing
systems including
[ICU MessageFormat](https://unicode-org.github.io/icu/userguide/format_parse/messages/) and [Fluent](https://projectfluent.org/).

The implementation of Fluent within Firefox mostly relies upon a declarative syntax in the
DOM, but it does provide an [API](https://firefox-source-docs.mozilla.org/l10n/fluent/tutorial.html#non-markup-localization)
for retrieving messages directly from Fluent when not being used to localize the DOM.

## Implementations

### Polyfill/transpiler implementations

The MessageFormat 2.0 specification is under development. An experimental implementation of one
proposal for the MessageFormat 2.0 specification is available at
[mf2](https://github.com/messageformat/messageformat/tree/mf2/packages/messageformat)
