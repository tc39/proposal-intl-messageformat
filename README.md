# Intl.MessageFormat

## Status

Champions: Eemeli Aro (Mozilla), Daniel Minor (Mozilla)

### Stage: 0

## Motivation

Intl.MessageFormat will build upon the [MessageFormat 2.0](https://github.com/unicode-org/message-format-wg/) (hereafter MF2.0)
specification currently under development. It will allow the use of MF2.0 resources
to localize web sites, enabling localization of the web using industry standard tooling and
processes. This should in turn make it easier to localize the web, increasing the openness
and accessibility of the web for speakers of languages other than the handful of languages for
which localization is typically done.

It is likely that eventually browsers themselves will be localized using MF2.0. This is already
planned for Firefox. If this happens, it will make sense to expose the MF2.0 implementation
already present in the browser to the web, rather than relying upon userland libraries.

## Use cases

* The primary use case is the retrieval of localized text ("a message") given a message
identifier and a previously specified locale.

## Description

### API

The MF2.0 specification is still being developed by the working group. The API below is based
upon one proposal under consideration, but should not be considered representative of a
consensus among the working group. In particular, the API shape of MessageFormatOptions,
Message and Scope will depend upon the data model chosen by the working group.

A `Resource` is a group of related messages for a single locale. Messages can be organized
in a flat structure, or in hierarchy, using paths. Conceptually, it is similar to a file
containing a set of messages, but there are no constrains implied on the underlying
implementation.

The interface provided by `Message` is part of the MF2.0 data model being developed by the
MF2.0 working group.

```
  interface Message { }

  interface Resource {
    getId(): string;

    getMessage(path: string[]): Message | undefined;
  }
```

The `Intl.MessageFormat` constructor creates `MessageFormat` instances for a given locale,
`MessageFormatOptions` and optional set of `Resource`s. The exact interface for
`MessageFormatOptions` is still under development by the MF2.0 working group. The `Scope`
provides lookup for variable references used for formatting. It is also still under
development by the working group.

```
  interface MessageFormatOptions { }

  interface Scope { }

  Intl.MessageFormat(
    locales: string | string[],
    options?: MessageFormatOptions | null,
    ...resources: Resource[]
  ) : MessageFormat;

  MessageFormat.addResources(...resources: Resource[]);

  MessageFormat.format(msgPath: string | string[], scope?: Scope): string;

  MessageFormat.format(resId: string, msgPath: string | string[], scope?: Scope): string;

  MessageFormat.formatToParts(
    resId: string,
    msgPath: string | string[],
    scope?: Scope
  ): MessageFormatPart[];

  MessageFormat.resolvedOptions() : object;
```

## Comparison

The MF2.0 specification is being developed based upon lessons learned from existing
systems including
[MessageFormat](https://unicode-org.github.io/icu/userguide/format_parse/messages/) and [Fluent](https://projectfluent.org/).

The implementation of Fluent within Firefox mostly relies upon a declarative syntax in the
DOM, but it does provide an [API](https://firefox-source-docs.mozilla.org/l10n/fluent/tutorial.html#non-markup-localization)
for retrieving messages directly from Fluent when not being used to localize the DOM.

## Implementations

### Polyfill/transpiler implementations

The MessageFormat 2.0 specification is under development. An experimental implementation of one
proposal for the MessageFormat 2.0 specification is available at
[mf2](https://github.com/messageformat/messageformat/tree/mf2/packages/messageformat)
