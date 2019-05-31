#  Patches proposal

This is a proposal for a patch system for TyD. Patches allow data to modify other data that was previously loaded.

These would be for use by:

* Localizations
* Mods
* Expansion packs

Conceptually, a patch can be though of as a sort of line in a script. Each patch carries out one operation on the data, modifying it in some specific way. Thus order of the patches can be important.

We had two patch-like systems in RimWorld: One for localization, and a separate one for modding. The localization system was relatively clean, but could have been made more concise and easy to use. The modding system worked, but was very verbose - it was based on XML's XPath system. This proposal attempts to do better than both, in one syntax.

Comments are welcome, on this proposal and all variants.

## Basics

The format for each patch is:

    PATH_TO_NODE OPERATOR (VALUE)

## Node paths

A node path is a sequence of *path actions*, each of which changes which nodes are selected. At any given step in the path, one or more nodes can be selected. Initially, the "invisible root" node is selected. This is the anonymous, implied table parent of all root nodes.

The path actions are:

* `/CHILDNAME`: From each selected node, select all children with a given name. This only works for tables.
* `@INDEX`: From each selected node, select child at index `INDEX`. If `INDEX` is negative, it counts backwards from the last child node; -1 is the last list element. `-0` is accepted as an index after the last child node and can be used for insertions operations. This works for both lists and tables.
* `/GRANDCHILDNAME="VALUE"`: From each selected node, select all children which themselves have a string child named `GRANDCHILDNAME` with value `VALUE`. Quotation marks are required around the value.
* `:CHILDNAME="VALUE"`: Filter selected nodes down to those with a string child named `CHILDNAME` with value `VALUE`.

## Operations

After you select a set of nodes, you want to transform it somehow. These are the operations that can change the selected node.

* `> NEW_VALUE`: Replace all selected nodes' values with `NEW_VALUE`. Value can be a string, null, table, or list, so this operation can change the selected nodes to a new type of node.
* `^ NEW_NODE`: Insert `NEW_NODE` before all selected nodes, as a child of its parent.
* `~`: Delete selected nodes.

## Examples

    # Select all nodes named "PlantDef", and change their children named "flammability" to have a value of 0.85
    /PlantDef/flammability > 0.85

    # Select all nodes named PlantDef with child string "id" of value "Wheat" and replace their "growSpeed" values
    /PlantDef:id="Wheat"/growSpeed > 5

    # Select the second of all Goblins' attacks and change their labels to "Crush face"
    /Goblin/attacks@2/label > "Crush face"

    # Select any of any Goblins' attacks which have the label "Facecrush", and change their labels to "Crush face"
    /Goblin/attacks(label "Facecrush")/label > "Crush face"

    /Goblin/attacks@0  ^ {...}  # Insert new node before index 0 (the first entry)
    /Goblin/attacks@-0 ^ {...}  # Insert new node before index -0 (after the last entry). New value ends up at the end
    /Goblin/attacks@-1 ^ {...}  # Insert new node before index -1 (the last entry)
    /Goblin/attacks@1  ~        # Delete the second entry
    /Goblin/attacks@-1 ~        # Delete the last entry

## Scope stack

Sometimes you want to patch many nodes with similar paths. A naive implementation of this can mean writing much of the same path over and over. Imagine we're writing a mod that partly redesigns one of the abilities of a particular enemy:

    /Goblin:id="ShamanLord"/abilities/attacks:label="ShadowBolt"/label > "Bolt of shadows"
    /Goblin:id="ShamanLord"/abilities/attacks:label="ShadowBolt"/secondaryEffects:type="Explosion"/radius > 5
    /Goblin:id="ShamanLord"/abilities/attacks:label="ShadowBolt"/secondaryEffects:type="Explosion"/damage > 25
    /Goblin:id="ShamanLord"/abilities/attacks:label="ShadowBolt"/secondaryEffects:type="Explosion"/castTime > 16
    /Goblin:id="ShamanLord"/abilities/attacks:label="ShadowBolt"/secondaryEffects:type="Explosion"/cooldown > 8
    /Goblin:id="ShamanLord"/abilities/attacks:label="ShadowBolt"/secondaryEffects:type="Explosion"/friendlyFire > false
    /Goblin:id="ShamanLord"/abilities/attacks:label="ShadowBolt"/secondaryEffects:type="Explosion"/visualEffect > DarkBlast

The above is cumbersome to read and probably slow to parse. To solve this problem, TyD has a system called *scope*. Using scope, the above can be written as:

    /Goblin:id="ShamanLord"/abilities/attacks:label="ShadowBolt"
    {
        /label       > "Bolt of shadows"

        /secondaryEffects:type="Explosion"
        {
            /radius          > 5
            /damage          > 25
            /castTime        > 16
            /cooldown        > 8
            /friendlyFire    > false
            /visualEffect    > DarkBlast
        }
    }

All node paths are interpreted as starting at the top node of the current *scope stack*. When the scope stack is empty, the invisible root is used as its top.

Note that since multiple nodes can be selected at once, each entry in the scope stack represents a group of nodes.

These scope actions modify the scope stack:

* `{`: Push the selected tables onto the scope stack. (If any selected node isn't a table, raise an error.)
* `}`: Pop a table group off the scope stack into the selection. (If the top of the stack isn't tables, raise an error.)
* `[`: Push the selected lists onto the scope stack. (If any selected node isn't a list, raise an error.)
* `]`: Pop a list group off the scope stack into the selection. (If the top node of the stack isn't lists, raise an error.)

## Examples

    # Scope into the root nodes named PlantDef with a child of name 'id' and value 'Wheat'
    /PlantDef:id="Wheat"
    {
        # Replace two of the values of named children
        /growSpeed > 5
        /minFerility > 3

        # Replace the first three entries of its child list named 'soilTypes'
        /soilTypes
        [
            @0 > SandySoil
            @1 > WetSoil
            @2 > MarshySoil
        ]
    }

    # Scope into all root nodes named TerrainDef
    /TerrainDef
    {
        #Modify two specific TerrainDef nodes, by filtering each one separately according to "id"
        :id="Soil"/fertility  > 8
        :id="Marsh"/fertility > 4
    }

    # Scope into nodes named 'Goblin'
    /Goblin
    {
        # Scope into nodes named 'attacks'
        /attacks
        [
            # This new value will be inserted as the new first entry
            @0 ^ {...whatever...}

            # The node at index 1 will be deleted
            # Note that since we just inserted at index 0, the new index 1 will be the original index 0
            # So this will delete the first entry from the original list
            # It would probably be best to put this first
            @1 ~

            # Select the second entry of this list, then replace the value of its first child named 'label'
            @2.label > "Crush face"

            # Select the first child of this list which has a child with name "label" and value "Facecrush"
            # Then replace the value of its first child named 'label'
            (label "Facecrush").label   > "Crush face"

            # Insert a new table node before current index 1
            @1 ^ {...whatever...}

            # Delete entry at current index 5
            @5 ~
        ]
    }

## Example - Localization

An realistic example of what localization might look like. This would probably be in a non-English language for a game with its primary language in English, but a developer could choose to put their English language data in patches separate from the gameplay data as well.

    /FactionDef:id="PirateBand"
    {
        /label          > "Pirate band"
        /description    > "An evil pirate band."
        /leaderTitle    > "Boss"
        /memberNames
        [
            @0 > "Evil Joe"
            @1 > "Big Nose Lenny"
            @2 > "Machine Gun Martha"
        ]
        greetingsDialogueSequence
        [
            @0/text > "I just have one question for you."
            @1/text > "Do you feel lucky?"
            @2/text > "Well, do you, punk?"
        ]
    }

Note that above, the greetingsDialogueSequence is a list of tables, and each table has a record named `text`, plus other game data. That `text` node's value is what we're replacing, while we leave the rest of the node untouched since it contains game data.

The below is legal but you don't want to do it in localization! Here we replace each entire list record with a new one that has the new `text`. But critically, this will also replace the rest of the record as well, so if there was any game data in there, it's gone now. This is an example of how a translator could destroy game data through careless patching.

    # Do not do this! Localization erasing game data.
    greetingsDialogueSequence   
    [
        @0 > {text "I just have one question for you."}
        @1 > {text "Do you feel lucky?"}
        @2 > {text "Well, do you, punk?"}
    ]

## Discussion

You can freely mix patches and declarations in the same file. The parser can distinguish patches from data because patch paths must start with a path action, and all path actions start with characters that are illegal as TyD node names. So, the TyD parser can easily know, from the first character, when it's reading a patch instead of data. This'll make parsing faster.

## Ideas

### Ideas - General

* `:CHILDNAME="VALUE"`: May be better to do this as `:/CHILDNAME="VALUE"`, and also allow other structures like `:@2="VALUE"`. This means `:` is more of a generalized "filter" operator that can be followed by all manner of tests, and the tests themselves use the same path syntax as anywhere else. We can also use operators like `|` and `&` and so on, parent addressors like `..`, tests besides `=` like `>` and `<` and `!=`, etc.

* If I do the above, I may also want to change `@2` to `/@2`, where `@2` is a stand-in for node name, and `/` is a generalized "select into" operator that selects children of current selection based on some filter. So all path actions thus start as either `/` select from children, or `:` select from current selection. One could even imagine these operators by themselves, such that `/` just means 'select all children', so the current `/` would have to be replaced by `/:` - 'select all children, then filter them'. That has a nice purity to it, but it's a bit verbose.

* Mixed data and patch: It would be great if patch and data syntax could be totally unified, so that the data syntax actually means "Add these records in this order and structure", and can be re-used as part of patching as desired. This also automatically supports things like "dotted paths" in data records. E.g if you just want `Goblin{attackDamage 7}` you can just write `Goblin.attackDamage 7'. TOML has a similar feature. This basically would just mean replacing the ^ insert operator with nothing at all.

* A way to load data in "any", "data only", or "patches only" mode. This prevents certain types of mistakes, like translators accidentally defining new content instead of patching existing content.

* A way to enforce localization patches to only be able to modify certain data. This may have to happen outside TyD itself since it depends on context knowledge from the game.

### Ideas on scoped patches

* Add an optional `?` character to operators, to mean 'optional'. If the node isn't found, an optional patch will be silently ignored, whereas a standard patch would generate an error.

* Ability to select the nth record with a given name. Otherwise it's quite tricky to address multiple records with the same names, since you need to use index alone.

* Custom or exotic operator syntax. We use the below syntax to allow defining either custom patch operator, or more exotic patch operators. So patches can do weird game-specific things like, say, capitalize all text, offset numbers by a certain amount, and so on.

    &OPERATOR_NAME OPERATOR_DATA

* Some way to select and affect multiple nodes at once.

* Selection by handle attribute:

    (*handle value)     Select a node by its handle
    PawnKind(*handle GoblinBase).speed > 25    # Replace a value in a def based on its handle

* Allow un-quoted strings in the `/NAME=VALUE` path action.

### Idea - Implied pathing in lists

This is a special rule for pathing to children of lists while in list scope.

While in the scope of a list, paths which don't start with an @ index are implied to start with an @ index for the nth entry in the list (as it currently exists in the patch sequence).

This means a patch can replace a whole list by simply listing a sequence of `>` or `~` operators one by one, in the same order as the original list.

So

    [
        > "Evil Joe"
        > "Big Nose Lenny"
        > "Machine Gun Martha"
    ]

Is interpreted as 

    [
        @0 > "Evil Joe"
        @1 > "Big Nose Lenny"
        @2 > "Machine Gun Martha"
    ]

And this:

    [
        /text > "I just have one question for you."
        /text > "Do you feel lucky?"
        /text > "Well, do you, punk?"
    ]

Is interpreted as:

    [
        @0/text > "I just have one question for you."
        @1/text > "Do you feel lucky?"
        @2/text > "Well, do you, punk?"
    ]

Unaddressed operators must come before all addressed operators in a list, otherwise an error is raised.

This could get messy when dealing with insertions.