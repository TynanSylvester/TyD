# Patch system proposal

This is a proposal for a patch system for TyD. Patches modify data that was previously declared. They are used to implement localizations, mods, and expansion packs.

Comments on this proposal are welcome.

## Basics

The format for a patch is:

    (PREFIX)TPATH OPERATOR VALUE

`TPATH` selects nodes we want to patch. `OPERATOR` and `VALUE` define how we want to change those nodes. `PREFIX` can be either `?` for optional, or nothing.

A patch which selects no nodes at all will generate an error, unless the patch is prefixed by the `?` question mark character.

    @Goblin/weapon : spear   # Throws an error if Goblin is missing
    ?@Goblin/weapon : spear  # No error even if Goblin is missing

Patches and data declarations can be freely mixed in the same file. Patches are applied relative to all previously-loaded declarations and previously-applied patches. This means that a patch must be loaded after the data that it is intended to modify.

    #Declare a record
    Goblin
    {
        id      GoblinWarlord
        weapon  Sword
    }

    # Patch the record to change the sword to a spear
    @Goblin/weapon : spear

There's no reason to declare and patch the same record in one file; it's easier to just change the original record. However, one may want to declare records and patch other records in other files, all in one file. In general, the boundaries of files are transparent to TyD.

## TPath

TPath is a system for selecting sets of nodes. It used both for selecting nodes to patch, and for selecting nodes to check for conditionals inside a TPath. (It may also be used for other purposes, like selecting nodes through an API.)

A TPath must begin with the `@` character. TPaths are made up of a sequence of path commands separated by the `/` forward slash. There should be no leading or trailing forward slashes. Whitespace inside TPaths is ignored.

    # These three paths are the same
    @Goblin/attacks/2

    @Goblin / attacks / 2
    
    @Goblin
        /attacks
            /2

Path commands consist of a target set marker, followed by a filter. The result of the command is all the nodes in the target set which satisfy the filter.

Target set markers are:

* If no target set marker appears, the target set is all children of all selected nodes.
* `.`: The target set is all selected nodes.
* `..`: The target set is the parents of all selected nodes.

There are several filters. They can also be logically composed together. The filters are:

* `NAME`: Filter the current selection to nodes with a given name. `*` works as a wildcard, alone or with text. When `*` is alone, it even selects anonymous nodes.
* `INDEX`: Written as a number like `0` or `15`. From each contiguous sequence of selected siblings, filter to only the Nth node.
* `TPATH=VALUE` and `TPATH!=VALUE`: Filter selected nodes to those that pass a given test. The test is: Any child selected by a `TPATH` starting the node being tested must have value exactly matching/not exactly matching `VALUE`. `VALUE` can be a string, a null, a list or a table. If `VALUE` is a string, and it contains any of `\/!:@<>+-^~|&` or whitespace, it must be quoted.

Filters can be composed with `&` (and) and `|` (or). Composed filters are resolved left-to-right. Filters can be grouped with `(...)`. Filters can be negated if preceded by `!`.

When writing patches, TPaths begin at the 'document parent', an abstract root node which is the parent of all root nodes in all documents being patched.

### TPath examples

    # Select all root nodes
    @*

    # Select all children of all root nodes
    @*/*

    # Select all nodes named 'id' which are children of any root node
    @*/id

    # Select all root nodes whose name includes the string 'Alcohol'
    @*Alcohol*

    # Select all root nodes named 'Goblin'
    @Goblin

    # Select all root nodes not named 'Goblin'
    @!Goblin

    # Select all nodes named 'id' which are children of root nodes named 'Goblin'
    @Goblin/id

    # Select all nodes which are children of root nodes named 'Goblin' except those named 'id' or 'attacks'
    @Goblin/!id & !attacks

    # Select the 3rd attack of every Goblin
    @Goblin/attacks/2

    # Select all root nodes named Goblin, then filter them down to those with a child named 'id' with value 'Shaman'
    @Goblin & @id=Shaman

    # Select all root nodes with a child named 'attitude' with value 'enemy'
    @attitude=enemy

    # Select all children of root nodes which are named 'attitude' with value 'enemy'.
    @* / attitude & @.*=enemy

    # Select all children of root nodes named Goblin which have the value 'blue'
    @Goblin / @.*=blue

    # Select all root nodes named Goblin, and with a child named 'warCry' with string value 'Attack!'
    @Goblin & @warCry="Attack!"

    # Select all goblins whose second weapon does ice damage
    @Goblin & @weapons/1/damageType=Ice

    # Select all goblins which are red or purple
    @Goblin & (@color=red | @color=purple)

    # Select all plant types which grow on sandy soil (e.g. any child of soilTypes has a value of 'SandySoil')
    @PlantDef & @soilTypes/*=SandySoil

    # Select all root nodes which have a child named 'species' with value 'goblin', and a child name 'weapon' with value 'axe'
    @@species=goblin & @weapon=axe

    # Select the color of all axe-wielding goblins and orcs. Species is defined by a 'species' record.
    @(@species=goblin | @species=orc) & @weapon=axe /color

    # Select the color of all axe-wielding goblins and orcs. Species is defined by record name.
    @Goblin | Orc & @weapon=axe /color

    # Select all Goblins that can cast magic missile or fireball
    @Goblin & (@spells/*=MagicMissile | @spells/*=Fireball)

    # Select all Goblins that use axe or spear, and all Orcs that use axe or spear
    # Select all Goblins and Orcs that can cast magic missile or fireball
    @(Goblin | Orc) & (@spells/*=MagicMissile | @spells/*=Fireball)

    # Select all Goblins that can case magic missile or fireball, except the Berserker
    @Goblin & (@spells/*=MagicMissile | @spells/*=Fireball) & @id!=Berserker

    # Select all Goblins that are blue, except those that can cast IceBolt
    @Goblin & @color=blue & !(@spells/*=IceBolt)

## Operations

After you select a set of nodes, you want to transform it somehow. These are the operations that can change the selected node.

* `: NEW_VALUE`: Replace all selected nodes' values with `NEW_VALUE`. Value can be a string, null, table, or list, so this operation can change the selected nodes to a new type of node. Note that if the original node is absent, nothing will be selected, so this will do nothing.
* `^ NEW_NODE`: Insert `NEW_NODE` before all selected nodes, as a child of its parent.
* `~`: Delete selected nodes.

### Basic patch examples

    # Select all nodes named "PlantDef", and change their children named "flammability" to have a value of 0.85
    @PlantDef/flammability : 0.85

    # Select all nodes named PlantDef with child string "id" of value "Wheat" and replace their "growSpeed" values
    @PlantDef & @id="Wheat"/growSpeed : 5

    # Select the second of all Goblins' attacks and change their labels to "Crush face"
    @Goblin/attacks/1/label : "Crush face"

    # Select any of any Goblins' attacks which have the label "Facecrush", and change their labels to "Crush face"
    @Goblin/attacks/@label="Facecrush"/label : "Crush face"

    @Goblin/attacks/0  ^ {...}  # Insert new node before index 0 (the first entry)
    @Goblin/attacks/-0 ^ {...}  # Insert new node before index -0 (after the last entry). New value ends up at the end
    @Goblin/attacks/-1 ^ {...}  # Insert new node before index -1 (the last entry)
    @Goblin/attacks/1  ~        # Delete the second entry
    @Goblin/attacks/-1 ~        # Delete the last entry

## Scope stack

Sometimes you want to patch many nodes with similar paths. A naive implementation of this can mean writing much of the same path over and over. Imagine we're writing a mod that partly redesigns one of the abilities of a particular enemy:

    @Goblin & @id="ShamanLord"/abilities/attacks/@label="ShadowBolt"/label : "Bolt of shadows"
    @Goblin & @id="ShamanLord"/abilities/attacks/@label="ShadowBolt"/secondaryEffects/@type="Explosion"/radius : 5
    @Goblin & @id="ShamanLord"/abilities/attacks/@label="ShadowBolt"/secondaryEffects/@type="Explosion"/damage : 25
    @Goblin & @id="ShamanLord"/abilities/attacks/@label="ShadowBolt"/secondaryEffects/@type="Explosion"/castTime : 16
    @Goblin & @id="ShamanLord"/abilities/attacks/@label="ShadowBolt"/secondaryEffects/@type="Explosion"/cooldown : 8
    @Goblin & @id="ShamanLord"/abilities/attacks/@label="ShadowBolt"/secondaryEffects/@type="Explosion"/friendlyFire : false
    @Goblin & @id="ShamanLord"/abilities/attacks/@label="ShadowBolt"/secondaryEffects/@type="Explosion"/visualEffect : DarkBlast

The above is cumbersome to read and probably slow to parse. To solve this problem, TyD has a system called *scope*. Using scope, the above can be written as:

    @Goblin & @id="ShamanLord"/abilities/attacks/@label="ShadowBolt"
    {
        label               : "Bolt of shadows"

        secondaryEffects/@type="Explosion"
        {
            radius          : 5
            damage          : 25
            castTime        : 16
            cooldown        : 8
            friendlyFire    : false
            visualEffect    : DarkBlast
        }
    }

Note that since multiple nodes can be selected at once, each entry in the scope stack represents a group of nodes. TPaths are interpreted as starting at the top nodes of the current *scope stack*. When the scope stack is empty, the document roots are used as the TPath roots.

These scope actions modify the scope stack:

* `{`: Push the selected tables onto the scope stack. (If any selected node isn't a table, raise an error.)
* `}`: Pop a table group off the scope stack into the selection. (If the top of the stack isn't tables, raise an error.)
* `[`: Push the selected lists onto the scope stack. (If any selected node isn't a list, raise an error.)
* `]`: Pop a list group off the scope stack into the selection. (If the top node of the stack isn't lists, raise an error.)

### Patch examples using scope

    # Scope into the root nodes named PlantDef with a child of name 'id' and value 'Wheat'
    @PlantDef & @id=Wheat
    {
        # Replace two of the values of named children
        growSpeed   : 5
        minFerility : 3

        # Replace the first three entries of its child list named 'soilTypes'
        # And delete the 4th entry
        soilTypes
        [
            0 : SandySoil
            1 : WetSoil
            2 : MarshySoil
            3 ~
        ]
    }

    # Scope into all root nodes named TerrainDef
    @TerrainDef
    {
        #Modify the fertility of two specific TerrainDef nodes, by filtering each one separately according to "id"
        @id="Soil" /fertility : 8
        @id="Marsh"/fertility : 4
    }

    # Scope into nodes named 'Goblin'
    @Goblin
    {
        # Scope into nodes named 'attacks'
        attacks
        [
            # This new value will be inserted as the new first entry
            0 ^ {...whatever...}

            # The node at index 1 will be deleted
            # Note that since we just inserted at index 0, the new index 1 will be the original index 0
            # So this will delete the first entry from the original list
            # It would probably be best to put this first
            1 ~

            # Select the second entry of this list, then replace the value of its first child named 'label'
            1/label : "Crush face"

            # Select the first child of this list which has a child with name "label" and value "Facecrush"
            # Then replace the value of its first child named 'label'
            @label="Facecrush"/label : "Crush face"

            # Insert a new table node before current index 1
            1 ^ {...}

            # Delete entry at current index 5
            5 ~
        ]
    }

## Example - Localization

An realistic example of what localization might look like. This would probably be in a non-English language for a game with its primary language in English, but a developer could choose to put their English language data in patches separate from the gameplay data as well.

    @FactionDef & @id="PirateBand"
    {
        label          : "Pirate band"
        description    : "An evil pirate band."
        leaderTitle    : "Boss"
        memberNames
        [
            0 : "Evil Joe"
            1 : "Big Nose Lenny"
            2 : "Machine Gun Martha"
        ]
        greetingsDialogueSequence
        [
            0/text : "I just have one question for you."
            1/text : "Do you feel lucky?"
            2/text : "Well, do you, punk?"
        ]
    }

Note that above, the greetingsDialogueSequence is a list of tables, and each table has a record named `text`, plus other game data. That `text` node's value is what we're replacing, while we leave the rest of the node untouched since it contains game data.

The below is legal but you don't want to do it in localization! Here we replace each entire list record with a new one that has the new `text`. But critically, this will also replace the rest of the record as well, so if there was any game data in there, it's gone now. This is an example of how a translator could destroy game data through careless patching.

    # Do not do this! Localization erasing game data.
    @greetingsDialogueSequence
    [
        0 : {text : "I just have one question for you."}
        1 : {text : "Do you feel lucky?"}
        2 : {text : "Well, do you, punk?"}
    ]

## Discussion

We had two separate patch systems in RimWorld: One for localization, and one for modding. The localization system was relatively clean, but could have been made more concise and easy to use. The modding system worked, but was very verbose - it was based on XML's XPath system. This proposal attempts to do better than both, in one syntax.

Patches sometimes have to modify data that may or may not be present, or may be in one of several forms. For example, if a mod wants to support multiple game versions, or wants to patch data in other mods. The patch system has to be flexible enough to support such cases.

## Ideas

* Ability to select the nth record with a given name. Otherwise it's quite tricky to address multiple records with the same names, since you need to use index alone. This may just want a jump operator that jumps to the currently-selected nodes.

* More filters using mathematical inequalities `<`, `>`, `<=`, `>=`. These would just parse the value as a decimal and do the comparison, throwing an error if a non-decimal value is found.

* Custom extensible operator syntax. We use the below syntax to allow defining either custom patch operator, or more exotic patch operators. So patches can do weird game-specific things like, say, capitalize all text, offset numbers by a certain amount, and so on.

    %CUSTOM_OPERATOR_NAME OPERATOR_DATA

* Add a filter by attribute (possibly including arbitrary attributes):

    *handle=value                              #Select a node by its handle
    PawnKind & *handle=GoblinBase/speed : 25  # Replace a value in a def based on its handle

* Implied pathing in lists: This is a special rule for pathing to children of lists while in list scope.

While in the scope of a list, paths which don't start with an index are implied to start with an index for the nth entry in the list (as it currently exists in the patch sequence).

This means a patch can replace a whole list by simply listing a sequence of `>` or `~` operators one by one, in the same order as the original list.

So

    [
        : "Evil Joe"
        : "Big Nose Lenny"
        : "Machine Gun Martha"
    ]

Is interpreted as 

    [
        0 : "Evil Joe"
        1 : "Big Nose Lenny"
        2 : "Machine Gun Martha"
    ]

And this:

    [
        /text : "I just have one question for you."
        /text : "Do you feel lucky?"
        /text : "Well, do you, punk?"
    ]

Is interpreted as:

    [
        0/text : "I just have one question for you."
        1/text : "Do you feel lucky?"
        2/text : "Well, do you, punk?"
    ]

Unaddressed operators must come before all addressed operators in a list, otherwise an error is raised.

This could get messy when dealing with insertions.