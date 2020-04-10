# TyD

Tynan's Tidy Data Language

By Tynan Sylvester

This project is not yet considered stable. The current version is 0.3.4.

You can use [TyDSharp, a simple C# implementation of the TyD language.](https://github.com/TynanSylvester/TyDSharp)

## Purpose

TyD is an easy-to-edit text data language designed for:

* **Game data** like enemy types, spells, terrain types, and so on.
* **User config data** like screen resolution, difficulty, and user accounts.

Since Tynan is an indie game developer (known for RimWorld), TyD was created with indie games in mind. However, it could be used for other types of software as well.

## Example

Below is a definition of a chess table object for a video game.

    ThingDef *source FurnitureBase
    {
        name                    ChessTable
        label                   "chess table"
        description             "The ancient game of kings. It trains intellectual skills."
        techLevel               Medieval
        researchPrerequisites   [ Woodworking; Varnish; Chess ]
        buildMaterialAmount     70
        buildMaterialTypes
        [
            Metal
            Wood
            Stone
            Glass
        ]
        stats
        {
            MaxHitPoints        100
            Mass                5
        }
    }

## Comparisons

Here's how TyD compares with some similar languages.

* JSON is designed for efficient machine interchange, with inflexible syntax and no comments. TyD is designed for interaction between human and machine, with more flexible syntax.
* XML is verbose and very general. TyD is brief and made to excel at its specific purposes only.
* TOML defines the type of every record in every document, and its syntax can be confusing. TyD simplifies editing by assuming most type info is known.
* YAML is very complex and very general, requiring a lot of code to parse properly. TyD is minimal, simple, and hard to make mistakes with.
* LUA is a full scripting language, which invites errors. Its syntax is not as clean as it could be for data definition.
* INI files can't express more complex data structures.

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

The name comes first, followed by some whitespace or newlines, followed by the value. After the value comes the record end, which is either a newline or a semicolon `;`. After the record end, a new record can begin on the same line. Multiple record ends in a row are allowed; the empty records are ignored.

    name1 value1
    name2        value2           # As much whitespace as you want between name and value
    name3                         # Value can be on another line after name
      value3
    name4 value4; name5 value5    # Multiple records on one line, separated by semicolon
    ;;;name6 value6;;name7 value7 # Multiple semicolons in a row are ignored

There are four types of record: String, null, list, and table (explained below).

#### Record names

A record's name can contain any alphanumeric character, or the underscore (`a-zA-Z0-9_`). Names must contain at least one alphabetical character (`a-zA-Z`).

    player_view_distance    50
    enemy7Speed             70

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

### Vertical strings

In vertical strings, each line begins with a vertical bar `|` character. A vertical string is initiated when the first non-whitespace character in the record value is `|`, and continues as long as the first character in each line is `|`.

Within a vertical string, characters are handled literally. There are no escape codes, and anything at all can be written.

These can be used to write multi-line strings in a visually pleasing format, while respecting the indentation of the rest of the file.

Example:

    example1    |This is my vertical string. It can have empty lines, like this:
                |
                |The beginning of each line is just after vertical bar. They can be misaligned, like this:
            |Though misaligned lines are harder to read, they make the format less pedantic in case things are being reformatted.
                |
                |It continues until there is a line without a | as first char.
    example2    |A new record was started, so this text is part of it instead of the example1.
    example3    |And this is another new record.

### Null

A null value is expressed with the naked string `null` (lowercase only). If you want the literal string "null", you need to use a quoted string.

    last_name           Null    # This person's name is actually Null.
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

### Inheritance

TyD supports inheritance between records. This reduces the need to repeat the same data in similar records.

For example, if you have five types of goblin enemies, you can define a single `BaseGoblin` record holding common info on all goblins like character model, skin, size, speed, and attack types. You can then have five concrete goblin records inherit from `BaseGoblin`, only varying their color and damage.

Inheritance is handled by the use of _attributes_ which can be attached to records. Attributes are defined after the record name and before the value. Each attribute declaration begins with an asterisk `*`.

The `*handle` attribute defines the record as having a given handle for the purposes of inheritance. It is followed by a string which defines the handle itself. The handle can contain the characters (a-zA-Z0-9_).

The `*source` attribute defines the record as inheriting data from the other record with the matching handle. It is followed by a string which defines the source's handle.

The `*abstract` attribute indicates that the record is abstract, which means it's only meant to be used as a base for inheritance, and not meant to be interpreted as data by itself. This means your code should not actually instantiate it. This attribute declaration appears alone, without any value after. TyD itself doesn't use this information; it's there for your code to help ignore records that are only needed as inheritance parents. If the abstract attribute is defined on a record, a handle attribute must also be defined, otherwise an error is thrown.

The `*noinherit` attribute indicates that the record should not inherit anything, even if its parent does so.

Records can only inherit from other records of the same type, with the exception of null records, which can participate in inheritance with any record.

If a string record has handle, source, or abstract attributes, an error should result.

#### How inheritance resolves

When a string or null inherits: Nothing happens.

When a list inherits: If the source is not null, the source's children are prepended to the heir's children.

When a table inherits: If the source is null, nothing happens. Otherwise, we check for matches between the names of children of source and heir. All unmatched children of source are prepended before heir's children. Then, the heir's first child of each name inherits from the source's first child of that name, according to these rules. (If multiple children have the same name, children after the first one are not affected by inheritance.)

#### Inheritance example

An example of inheritance:

    PlantType *handle BasePlant *abstract  # An abstract base plant type
    {
        height          10
        growthRate      25
        allowedSoilTypes
        [
            RichSoil
        ]
    }

    PlantType *source BasePlant            # A concrete plant type inheriting from BasePlant
    {
        name            Potato
        nutrition       1000
        growthRate      15                 # Override the value from BasePlant
        allowedSoilTypes
        [
            StonySoil                      # The final potato plant will allow both RichSoil and StonySoil
        ]
    }

## Contributing to TyD

All contributions including documentation, pull requests, and bug reports are welcome!

Potential future developments are listed on the [potential future developments page](https://github.com/tyd-lang/TyD/blob/master/Future.md).
