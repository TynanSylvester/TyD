# Patches proposal B - Scoped patches

## Basics

This is an add-on to the 'pathed patches' proposal.

Scoped patches solve the problem where you want to affect many sibling nodes, and so are forced to write many similar paths over and over. With scoped patches, you can just enter the scope of the common path, and then address the leaf nodes separately.

## Pathing rules

All paths are interpreted as starting at the top node of the current *scope stack*. At start, the scope stack is empty, so the invisible root is used.

We add these new path actions:

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

## Ideas

### Implied pathing in lists

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