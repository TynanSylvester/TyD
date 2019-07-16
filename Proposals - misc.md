# Proposed developments

The following features are proposed to add to TyD. These are roughly in priority order.

## Patches

A way to load TyD data that modified previously-loaded TyD data. Used for localization, user mods, and official expansion packs.

[More info here.](https://github.com/tyd-lang/TyD/blob/master/Proposal-patches.md)

## Literal strings

Literal strings are string format that begins and ends with the single quote `'` character. They must appear on one line. No escaping is allowed - what you see is exactly what you get. It's possible to write a literal string containing any character except the single quote itself.

 ## Multi-line literal strings

Multi-line literal strings start and end with `'''`. A newline immediately after the opening quotes will be trimmed.

Example:

    description '''
    This is my multi-line literal string. Note that the newline above is trimmed.
    
    It can include {} <> "" '' any char until it ends.'''

## Multi-line comments

A way to write comments on multiple lines with one start/end code, similar to C-style `/* comments */`. Suggested syntax is `#[ comment ]#`.

## Multiple inheritance

 A record can have multiple sources. It would inherit from them in the order they're declared, with each successive inhertiance overwriting data from the previous one the same way a record's own data overwrites that of its source.

## TyD VSCode extension

A VSCode extension for TyD that does syntax highlighting and navigation.

If we have patches, the VSCode extension for TyD should automatic revealing of addressed value when writing patches. This means translators can see the original text they’re translating, and modders can see the original value they’re modding, live in the text editor.

## Unicode char escape seqences in strings

A way to directly write escaped Unicode characters in strings. TOML has this feature.

## Line-ending backslash

Ending a line with a backslash `\` inside a string makes the parser ignore the newline. This allows more comfortably writing long strings in some cases.

## Arbitrary attributes

Support to add arbitrary attributes to records, which can be read by code and used according to the needs of a specific project. We may support this, though I'm also slightly doubtful given potential complexity and performance concerns.