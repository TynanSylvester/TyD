# TyD

Tynan's Tidy Data Format

By Tynan Sylvester

This project is not yet considered stable. The current version is 0.1.0.

You can use [TyDSharp, a simple C# implementation of the TyD format.](https://github.com/tyd-format/TyDSharp)

## Purpose

TyD is a simple text data format designed for:

* **Game data** like enemy types, spells, terrain types, and so on.
* **Config data** like resolution, difficulty, and user accounts.
* **Save data** like saved maps, characters, or inventory.

Since Tynan is an indie game developer (known for RimWorld), TyD was created with indie games in mind. However, it could be used for many other types of software as well.

## Example

Below is a definition of a chess table object for a video game.

    ThingDef *source FurnitureBase
    {
        name                    ChessTable
        label                   "chess table"
        description             "The ancient game of kings. It trains intellectual skills."
        techLevel               Medieval
        buildMaterialAmount     70
        buildMaterialTypes
        [
            Metal
            Wood
            Stone
        ]
        stats
        {
            MaxHitPoints        100
            Mass                5
        }
        researchPrerequisites   [ ComplexFurniture; Chess ]
    }

## Comparisons with similar formats

Here's how TyD compares with some similar formats.

* INI files are easy to hand-edit, but can't express more complex data structures.
* JSON is designed for efficient machine interchange, with inflexible syntax and no comments. TyD is designed for interaction between human and machine, with more flexible syntax.
* TOML defines the type of every record in every document. TyD simplifies editing by assuming most type info is known.
* YAML is complex and very general. TyD is minimal, simple, and hard to make mistakes with.
* XML is verbose and very general. TyD is brief and made to excel at its specific purposes only.
* LUA requires a heavy script interpreter and its scripting functionality is error-prone. TyD is lightweight and purpose-designed for data definition.

## Spec

* TyD is case sensitive.
* A TyD file must be a valid UTF-8 encoded Unicode document.
* Whitespace means tab (0x09) or space (0x20) or newline.
* Newline means LF (0x0A) or CRLF (0x0D0A).
* TyD files should use the extension `.tyd`.

### Comment

A hash `#` marks the rest of the line as a comment.

    # This is a full line comment
    playerViewHeight 1.85 # This is a comment at the end of a data line.

### Record

The core building block of a TyD document is the name/value pair. This pair is called a _record_.

The name comes first, followed by some whitespace or newlines, followed by the value.

    name1 value1
    name2        value2     # As much whitespace as you want between name and value
    name3                   # Value can be on another line after name
      value3

### Name

A name can contain any alphanumeric character, underscore, or hyphen (`a-zA-Z0-9_-`).

    player_speed        50
    enemy7Speed         70
    Jenny-Phone-Number  8675309
    1                   "The first level is very easy."

### String

All strings must contain only valid UTF-8 characters. Strings can contain any Unicode character with exceptions described below.

The below characters have TyD-defined escape sequences.

* `\\` Backslash               (U+005C)
* `\"` Double quote            (U+0022)
* `\#` Hash                    (U+0023)
* `\]` Right square bracket    (U+005D)
* `\}` Right curly bracket     (U+007D)
* `\;` Semicolon               (U+003B)
* `\r` Carriage return         (U+000D)
* `\n` Line feed               (U+000A)
* `\t` Tab                     (U+0009)

Use of any escaped character sequence (starting with `\`) besides the above should result in an error message.

There are two ways to express strings: _naked_ or _quoted_.

#### Naked strings

Naked strings begin with any character except `"`, `[`, `{`, or `*`. If you want a string to begin with any of these, you can't use a naked string.

Naked strings end with a newline, semicolon `;`, closing square bracket `]`, closing curly brace `}`, or hash `#`. Note that we must end on closing brackets because they can indicate the end of an enclosing collection, and we must end on hash because it indicates the start of a comment.

Whitespace is trimmed off the end of naked strings.

These characters must be escaped: `\` backslash, `#` hash, `;` semicolon.

    breed_name              Corgi
    description             A stubby dog that barks with a "yip" sound.\n\nMy friends don't like them\; I think they're \#1.
    length                  10.5cm; width 5.5cm; calories 800
    body_part_descriptions  { tail short; legs stumpy; tongue droopy and wet }

#### Quoted strings

Quoted strings begin and end with double quote marks `"`.

These characters must be escaped: `\` backslash, `"` double quote, `#` hash.

    dog_line_4 "The \#4 dog said, \"Arf!\"
    Then he jumped in my lap.
    
    Boy, I loved that dog."

TyD parsers should try to interpret newlines in a way that makes sense on their platform. On Linux, the newlines above would become `\n` characters, while on Windows they may become `\r\n` sequences.

### Null

A null value is expressed with the naked string `null` (lowercase only). If you want the literal string "null", you need to use a quoted string.

    last_name           "Null"  # This person's name is actually Null.
    criminal_record     null    # This person has no criminal record.
    favorite_word       "null"  # This person's favorite word is the word 'null'.

### List

Lists are ordered collections of anonymous records. Lists begin and end with square brackets `[]`. The records inside a list are _anonymous_ in that they have no names. All records in a list must be the same type (except null and string, which can be mixed).

    groceriesNeeded     # A list of strings
    [
        salt
        6 pears
        something to put on toast
    ]

    attackTypes         # A list of tables
    [
        {name fireball; cooldown 2.0; damage 7}
        {name magic missile; cooldown 1.0; damage 3}
    ]

### Table

Tables are collections of named records. Tables begin and end with curly brackets `{}`. Records in a table can be of different types.

    AnimalType      # A table called AnimalType, describing a type of animal
    {
        name                bear
        mass                800
        primaryAttack       { name bite;  damage 60 }
        secondaryAttack     { name claws; damage 40 }
        flyingMethod        null
        dietCategories
        [
            Meat
            Fish
            Berries
            Honey
        ]
    }

### Attributes

Each record can have some _attributes_. Attributes are defined after the name and before the value, with each attribute declaration beginning with an asterisk `*`.

The attributes are `handle`, `source`, and `abstract`, and `class`.

An example of a record with attributes:

    PlantType *handle BasePlant *abstract # An abstract base plant type
    {
        growthRate 25
    }

    PlantType *class TuberPlant *source BasePlant # A concrete plant type for a 
    {
        name Potato
        nutrition 1000
    }

#### Handle attribute

TyD supports inheritance relationships between records. This reduces the need to repeat the same data in many similar records. For example, if you have five types of goblin enemies, you can define a single `BaseGoblin` record holding common info on all goblins like character model, skin, size, speed, and attack types. You can then have five concrete goblin records inherit from `BaseGoblin`, only varying their color and damage.

The `handle` attribute defines the record as having a given handle for the purposes of inheritance.

This attribute declaration is followed by a string which defines the handle itself. The string can contain the characters (a-zA-Z0-9_-).

#### Source attribute

This defines the record as inheriting data from the other record with the matching handle.

This attribute declaration is followed by a string which defines the source's handle.

#### Abstract attribute

This adds a boolean indicating the record is abstract, which means your code should not actually instantiate it. It's only present to inherit from.

This attribute declaration appears alone, without any value after.

TyD itself doesn't use this information; it's there for your code to help ignore records that are only needed as inheritance parents.

#### Class attribute

This adds a string to the record that your code can use to disambiguate which type or class should be used when interpreting this record.

This attribute declaration is followed by a string which defines the actual name of the class. The string can contain the characters (a-zA-Z0-9_-).

TyD itself doesn't use this information for anything; it's there so you can attach data to help your code know how to handle the record.

## Contributing to TyD

All contributions including documentation, pull requests, and bug reports are welcome!

## Potential future developments

The following developments are under consideration to add to TyD. If you're interested in implementing any of them, please get in touch!

**Multi-line comments:** A way to write comments on multiple lines with one start/end code, similar to C-style `/* comments */`. Suggested syntax is `#[ comment ]#`.

**Multiple inheritance:** A record can have multiple sources. It would inherit from them in the order they're declared, with each successive inhertiance overwriting data from the previous one the same way a record's own data overwrites that of its source.

**Literal strings:** Strings which are delimited by single quotes, and don't allow any escaping whatsoever. Until the second `'` character, what you see is exactly what you get. This can also be extended to multi-line literal strings, denoted by `'''`, which allow including `'` characters. TOML has this feature.

**TyD VSCode extension:** A VSCode extension for TyD that does syntax highlighting and navigation.

**Unicode char escape seqences in strings:** A way to directly write escaped Unicode characters in strings. TOML has this feature.

**Line-ending backslash:** Ending a line with a backslash `\` inside a string makes the parser ignore the newline. This allows more comfortably writing long strings in some cases.

**Arbitrary attributes:** Users may want to add other attributes. We may support this, though I'm also slightly doubtful given potential complexity and performance concerns.

**Single-character attribute declarations:** Instead of declaring attributes with short strings like `source` or `handle`, we declare them with single special characters.

Examples:

    *Animal     # The asterisk defines the class of this record.
    $Bear       # The dollar sign defines the handle of this record for purposes of inheritance.
    ^Bear       # The up-arrow defines a source of this record for purposes of inheritance.
    ~           # The tilde means the record is abstract.

    AnimalType ~ ^BaseAnimal $BaseBear *AnimalWithClaws {...} # This record is named AnimalType. It is abstract. It inherits from BaseAnimal. Its handle is BaseBear. It uses class AnimalWithClaws.

**Patching:** Allow a document to patch another document, if loaded in a specific order using specific calls. For use in mods or expansion packs.

**Error recovery with log:** Provide a mode where instead of throwing exceptions and stopping upon finding errors in the text data, the system sends out log messages through a passed-in error message channel and continues with deserialization as best it can.
