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

A node path is a sequence of *path actions*, each of which moves the selection to a new node in some way relative to the currently selected node.

There is always exactly one node selected. Initially, the selected node is the "invisible root", which is the anonymous, implied parent of all the root nodes.

The path actions are:

* `/CHILDNAME`: Select the first child of the current node with a given name.
* `@INDEX`: Select the child of the current node at index `INDEX`. If `INDEX` is negative, it counts backwards from the last child node; -1 is the last list element. `-0` is accepted as an index after the last child node and can be used for insertions operations. This works for both lists and tables.
* `(GRANDCHILDNAME "VALUE")`: Select the first child of the current node which itself has a string child with the given name and value. Quotation marks are required around the value.

## Operations

After you select a node using a path, you want to transform it somehow. These are the operations that can change the selected node.

* `> NEW_VALUE`: Replace the selected node's value with `NEW_VALUE`. This can change the new type.
* `^ NEW_NODE`: Insert `NEW_NODE` before the selected node, as a child of its parent.
* `~`: Delete selected node.

## Examples

    # Select the node with a child node named "id" with value "Wheat" and replace its growSpeed
    (id "Wheat")/growSpeed > 5

    # Select the second of the Goblin's attacks and change its label to "Crush face"
    /Goblin/attacks@2/label > "Crush face"

    # Select whichever of the Goblin's attacks has the label "Facecrush" and change its label to "Crush face"
    /Goblin/attacks(label "Facecrush")/label > "Crush face"

    /Goblin/attacks@0  ^ {...}  # Insert new node before index 0 (the first entry)
    /Goblin/attacks@-0 ^ {...}  # Insert new node before index -0 (after the last entry). New value ends up at the end
    /Goblin/attacks@-1 ^ {...}  # Insert new node before index -1 (the last entry)
    /Goblin/attacks@1  ~        # Delete the second entry
    /Goblin/attacks@-1 ~        # Delete the last entry

## Scope stack

You don't need to write the full path to each node you want to patch, if you use scope.

All node paths are interpreted as starting at the top node of the current *scope stack*. At start, the scope stack is empty, so the invisible root is used.

These path actions interact with scope:

* `{`: Push the selected table onto the scope stack. (If the selected node isn't a table, raise an error.)
* `}`: Pop a table off the scope stack. (If the top node of the stack isn't a table, raise an error.)
* `[`: Push the selected list onto the scope stack. (If the selected node isn't a list, raise an error.)
* `]`: Pop a list off the scope stack. (If the top node of the stack isn't a list, raise an error.)

## Examples

    # Enter the first root node with a child of name 'id' and value 'Wheat'
    (id "Wheat")
    {
        # Replace two of the values of its named children
        /growSpeed > 5
        /minFerility > 3

        # Replace the first three entries of its first child list named 'soilTypes'
        /soilTypes
        [
            @0 > SandySoil
            @1 > WetSoil
            @2 > MarshySoil
        ]
    }

    # Enter the first node named 'Goblin'
    /Goblin
    {
        # Enter the first node named 'attacks'
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

    (id "PirateBand")
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

You can freely mix patches and declarations in the same file.

The parser can distinguish patches from data because patch paths must start with a pathaction, and all pathactions start with characters that are illegal as TyD node names. So, the TyD parser can easily know, from the first character, when it's reading a patch instead of data. This'll make parsing faster.

## Ideas

### Ideas - General

* Mixed data and patch: It would be great if patch and data syntax could be totally unified, so that the data syntax actually means "Add these records in this order and structure", and can be re-used as part of patching as desired. This also automatically supports things like "dotted paths" in data records. E.g if you just want `Goblin{attackDamage 7}` you can just write `Goblin.attackDamage 7'. TOML has a similar feature. This basically would just mean replacing the ^ insert operator with nothing at all.

* A way to load data in "any", "data only", or "patches only" mode. This prevents certain types of mistakes, like translators accidentally defining new content instead of patching existing content.

* A way to enforce localization patches to only be able to modify certain data. This may have to happen outside TyD itself since it depends on context knowledge from the game.

* Make a *patches<> context marker to mark patches. Anything that appears in between the <> angle brackets is parsed as a patch. This is a bit simpler than parsing every record individually, though it makes mixed data and patch impossible.

### Ideas on scoped patches

* Add an optional `?` character at the end of the path, to mean 'optional'. If the node isn't found, an optional patch will be silently ignored, whereas a standard patch would generate an error.

* Ability to select the nth record with a given name. Otherwise it's quite tricky to address multiple records with the same names, since you need to use index alone.

* Custom or exotic operator syntax. We use the below syntax to allow defining either custom patch operator, or more exotic patch operators. So patches can do weird game-specific things like, say, capitalize all text, offset numbers by a certain amount, and so on.

    &OPERATOR_NAME OPERATOR_DATA

* Some way to select and affect multiple nodes at once.

* Selection by handle attribute:

    (*handle value)     Select a node by its handle
    PawnKind(*handle GoblinBase).speed > 25    # Replace a value in a def based on its handle

* Allow un-quoted strings in the `(NAME VALUE)` path action.

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

