# TyD

Tynan's Tidy Data Format

By Tynan Sylvester

This project is not yet considered stable. The current version is 0.1.0.

## Purpose

TyD is a simple text data format designed for:

* **Game data** like enemy types, spells, terrain types, and so on.
* **Config data** like resolution, difficulty, and user accounts.
* **Save data** like saved maps, characters, or inventory.

Since Tynan is an indie game developer (known for RimWorld), TyD was created with indie games in mind. However, it could be used for many other types of software as well.

## Comparisons

Here's how TyD compares with some similar formats.

* INI files are easy to hand-edit, but can't express more complex data structures.
* JSON is designed for efficient machine interchange, with inflexible syntax and no comments. TyD is designed for interaction between human and machine, with more flexible syntax.
* TOML defines the type of every record in every document. TyD simplifies editing by assuming most type info is known.
* YAML is complex and very general. TyD is minimal, simple, and hard to make mistakes with.
* XML is verbose and very general. TyD is brief and made to excel at its specific purposes only.
* LUA requires a heavy script interpreter and its scripting functionality is error-prone. TyD is lightweight and purpose-designed for data definition.

## Examples

Below is a definition of a chess table object, including info on name, description, physical interaction, building requirements, UI keys, stats like health and mass, and visuals.

    ThingDef *source FurnitureWithQualityBase
    [
        defName                 ChessTable
        label                   "chess table"
        description             "The ancient game of kings. Fun for a few hours here and there, even playing alone. It trains intellectual skills."
        altitudeLayer           Building
        passability             PassThroughOnly
        fillPercent             40%
        pathCost                50
        rotatable               false
        socialPropernessMatters true
        minTechLevelToBuild     Medieval
        researchPrerequisites   { ComplexFurniture }
        stuffCategories         { Metallic; Woody; Stony }
        costStuffCount          70
        designationCategory     Joy
        designationHotKey       Misc2
        building
        [
            joyKind             Gaming_Cerebral
        ]
        statBases   #Note: We need to use [] since the elements have names, even thought these are a list in-game
        [           #Needs to be handld specially by the game code
            MaxHitPoints        100
            WorkToBuild         15000
            Mass                5
            Flammability        100%
            Beauty              4
            JoyGainFactor       1
        ]
        comps
        {
            *class CompProperties_RoomIdentifier
            [
                roomStat        Impressiveness
            ]
        }
        graphicData
        [
            texPath             Things/Building/Joy/ChessTable
            graphicClass        Graphic_Single
            shadowData          [ volume (0.65, 0.25, 0.6); offset (0, 0, -0.15) ]
            damageData
            [
                rect (0.09375, 0.296875, 0.796875, 0.703125)
                cornerTL Damage/Corner; cornerTR Damage/Corner; cornerBL Damage/Corner; cornerBR Damage/Corner
            ]
        ]
    ]

Below is a definition of a particular 'stat' that can be applied to any character. It defines the name and output style of the stat, as well as how it is calculated from the character's health capacities and skills.

    StatDef
    [
        defName         MiningSpeed
        label           "mining speed"
        description     "A speed at which this person digs at walls and drills for deep resources."
        category        PawnWork
        defaultBaseValue 1
        minValue        0.1
        toStringStyle   PercentZero
        statFactors
        {
            WorkSpeedGlobal
        }
        capacityFactors
        {
            [capacity Manipulation; weight 1]
            [capacity Sight; weight 0.5; max 1]
        }
        skillNeedFactors
        {
            *Class SkillNeed_BaseBonus
            [skill Mining; baseValue 0.04; bonusPerLevel 0.12]
        }
        scenarioRandomizable true
    ]

## Spec

* TyD is case sensitive.
* A TyD file must be a valid UTF-8 encoded Unicode document.
* Whitespace means tab (0x09) or space (0x20) or newline.
* Newline means LF (0x0A) or CRLF (0x0D0A).
* TyD files should use the extension `.tyd`.
* When transferring TyD files over the internet, use the MIME type `application/tyd`.

### Comment

A hash `#` marks the rest of the line as a comment. This is ignored inside a quoted string, but can terminate a naked string.

    # This is a full line comment
    playerViewHeight 1.85 # This is a comment at the end of a data line.

_Double slash may be changed to double hash later since double slash creates issues with file paths._

### Record

The core building block of TyD a TyD document is a name/value pair called a _record_.

The name comes first, followed by any non-zero amount of whitespace, followed by the value. Whitespace on a line after a value is ignored.

    name value
    name        value

The value does not need to start on the same line as the name.

    name
      value

### Name

A name can contain any alphanumeric character, underscore, or hyphen (`a-zA-Z0-9_-`).

    player_speed 500
    enemy7Speed 700
    Jenny-Phone-Number 8675309
    1 "The first level is very easy."

### String

There are two ways to express strings: _naked_ or _quoted_.

All strings must contain only valid UTF-8 characters. Strings can contain any Unicode character with exceptions described below.

The below characters have TyD-defined escape sequences.

* `\\` Backslash       (U+005C)
* `\"` Double quote    (U+0022)
* `\#` Hash            (U+0023)
* `\;` Semicolon       (U+003B)
* `\r` Carriage return (U+000D)
* `\n` Line feed       (U+000A)
* `\t` Tab             (U+0009)

Use of any escaped character sequence (starting with `\`) besides the above should result in an error message.

#### Naked strings

Naked strings begin with any character except `"`, `[`, `{`, or `*`. If you want a string to begin with any of these, you can't use a naked string.

Naked strings end with a newline, semicolon `;`, or hash mark `#`. Whitespace is trimmed off the end when parsing.

The characters below must be escaped.

* `\` Backslash
* `#` Hash mark
* `;` Semicolon

    breed_name              Corgi
    description             A sausage-shaped dog that barks with a "yip" sound.\n\nMy friends don't like them\; I think they're \#1.
    length                  10.5cm; width 5.5cm; calories 800
    body_part_descriptions  [ tail short; legs stumpy; tongue droopy and wet ]

#### Quoted strings

Quoted strings begin and end with double quote marks `"`.

The characters below must be escaped.

* `\` Backslash
* `"` Double quote
* `#` Hash mark

    dog_line_4 "The \#4 dog said, \"Arf!\"
    Then he jumped in my lap.
    
    Boy, I loved that dog."

TyD parsers should try to interpret newlines in a way that makes sense on their platform. On Linux, the newlines above would become `\n` characters, while on Windows they may become `\r\n` sequences.

### Null

A special null value is expressed with the naked string `null` (lowercase only). If you want the literal string "null", you need to use a quoted string.

    first_name          John
    last_name           "Null"
    criminal_record     null

### List

List record values begin with a curly brace `{` character. Following the curly brace come a list of _anonymous_ records in order. These are the same as normal records, except the name is omitted. The list ends with a matching curly brace `}`. Lists can contain strings, tables, or other lists - all anonymous. All records in a list must be the same type.

    groceriesNeeded
    {
        salt
        6 pears
        something to put on toast
    }

    attackTypes
    {
        [name fireball; cooldown 2.0; damage 7]
        [name magic missile; cooldown 1.0, damage 3]
    }

### Table

Tables begin with a square bracket `[` character. Following the square bracket are a list of named records in no particular order. The table ends with a matching square bracket `]`. Tables can contain any kind of record. Elements in a list will often vary in type.

    AnimalType
    [
        name bear
        mass 800
        primaryAttack    [ name bite;  damage 60 ]
        secondaryAttack  [ name claws; damage 40 ]
        dietCategories
        {
            Meat
            Fish
            Berries
            Honey
        }
    ]

### Attributes

Each record can have some _attributes_. Attributes are defined after the name and before the value, with each attribute declaration beginning with an asterisk `*`.

The attributes are `handle`, `source`, and `abstract`, and `class`.

An example of a record with attributes:

    PlantType *handle BasePlant *abstract # An abstract base plant type
    [
        growthRate 25
    ]

    PlantType *class TuberPlant *source BasePlant # A concrete plant type for a 
    [
        name Potato
        nutrition 1000
    ]

#### Handle attribute

TyD supports inheritance relationships between records. This reduces the need to repeat the same data in many similar records. For example, if you have five types of goblin enemies, you can define a single `BaseGoblin` record holding common info on all goblins like character model, skin, size, speed, and attack types. You can then have five concrete goblin records inherit from `BaseGoblin`, only varying their color and damage.

This defines the record as having a given handle for the purposes of inheritance.

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

    AnimalType ~ ^BaseAnimal $BaseBear *AnimalWithClaws [...] # This record is named AnimalType. It is abstract. It inherits from BaseAnimal. Its handle is BaseBear. It uses class AnimalWithClaws.

**Patching:** Allow a document to patch another document, if loaded in a specific order using specific calls. For use in mods or expansion packs.

**Error recovery with log:** Provide a mode where instead of throwing exceptions and stopping upon finding errors in the text data, the system sends out log messages through a passed-in error message channel and continues with deserialization as best it can.
