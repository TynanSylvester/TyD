# Potential future developments

The following developments are under consideration to add to TyD. If you're interested in implementing any of them, please get in touch!

## Patches

A way to load TyD data that modified previously-loaded TyD data. Used for localization, user mods, and official expansion packs.

## Multi-line comments

A way to write comments on multiple lines with one start/end code, similar to C-style `/* comments */`. Suggested syntax is `#[ comment ]#`.

## Multiple inheritance

 A record can have multiple sources. It would inherit from them in the order they're declared, with each successive inhertiance overwriting data from the previous one the same way a record's own data overwrites that of its source.

## Literal strings

 Strings which are delimited by single quotes, and don't allow any escaping whatsoever. Until the second `'` character, what you see is exactly what you get. This can also be extended to multi-line literal strings, denoted by `'''`, which allow including `'` characters. TOML has this feature.

## TyD VSCode extension

A VSCode extension for TyD that does syntax highlighting and navigation.

If we have patches, the VSCode extension for TyD should automatic revealing of addressed value when writing patches. This means translators can see the original text they’re translating, and modders can see the original value they’re modding, live in the text editor.

## Unicode char escape seqences in strings

A way to directly write escaped Unicode characters in strings. TOML has this feature.

## Line-ending backslash

Ending a line with a backslash `\` inside a string makes the parser ignore the newline. This allows more comfortably writing long strings in some cases.

## Arbitrary attributes

Support to add arbitrary attributes to records, which can be read by code and used according to the needs of a specific project. We may support this, though I'm also slightly doubtful given potential complexity and performance concerns.