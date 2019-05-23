#  Proposal - Patches

This document proposes a patch system for TyD. Patches allow data to modify other data that was previously loaded.

These would be for use by:

* Localizations
* Mods
* Expansion packs

Conceptually, a patch can even be though of as a sort of line in a script. Each patch carries out one operation on the data, modifying it in some specific way. Thus order of the patches is critical.

We had two patch-like systems in RimWorld: One for localization, and a separate one for modding. The localization system was relatively clean, but could have been made more concise and easy to use. The modding system worked, but was very verbose - it was based on XML's XPath system. This proposal attempts to do better than both, in one syntax.

## Basics

The format for each patch record is:

    ADDRESS OPERATION (VALUE)

The parser knows which records are patches based on their syntax.

## Node addressing

Node addressing starts from the root. You traverse the node tree via a number of operators. In each case, if there are multiple nodes satisfied by an operator, the system always chooses the one that came first (this is different from XML's XPath, which can select multiple nodes at once.)

    name                    # Select a node by its name.
    (childName value)       # Select a node by the by value of one of its named children.
    name(childName value)   # Select a node by its name combined with the value of one of its named children
    @number                 # Select a node by its index in the parent. Does not work for parent-less (root) nodes, but works for nodes with any parent, whether list or table. Works with negative numbers counting backwards from the end of the list; -1 is the last list element.

After selecting a node, you move into it with the `.` operator.

## Operations

After you address a node, you want to perform an operation on it. These define the operations. Note that these operators are the only way the parser can distinguish between patches and normal data in some cases.

    > value            Replace with value
    ^ value             Insert value
    ~                   Delete

The value can be a string, list, or table, in standard TyD syntax.

## Examples - Single value patches

    PlantDef(id Wheat).growSpeed             > 5             # Select the PlantDef with id "Wheat" and replace its growSpeed
    Goblin.attacks@2.label                   > "Crush face"  # Select a member of a list by index and replace it
    Goblin.attacks.(label "Facecrush").label > "Crush face"  # Select a member of a list by one of its values and replace it
    Goblin.attacks.@0                        ^ {...}         # Insert before index 0 (the first entry)
    Goblin.attacks.@1                        ^ {...}         # Insert before index 1 (the second entry)
    Goblin.attacks.@-0                       ^ {...}         # Insert before index -0 (after the last entry), so value ends up at the end
    Goblin.attacks.@-1                       ^ {...}         # Insert before index -1 (the last entry)
    Goblin.attacks.@1                        ~               # Delete the second entry
    Goblin.attacks.@-1                       ~               # Delete the last entry

## Examples - Multi value patches

In multi-value patches, address is generally the same, with one major exception: List entries without an address are matches to the nth entry in the original list, instead of always matching to `@-0`. This means you can just write a new list and it'll match the original one record-by-record.

    # Everything inside the braces is counted as though it came after what was outside the braces, plus .
    PlantDef-Wheat
    {
        growSpeed > 5
        minFertility > 3
    }
    
    PlantDef(name Wheat)
    {
        growSpeed > 5
        minFerility > 3
    }
    
    PawnType-Goblin
    {
        attacks
        [
            @2.label                    > "Crush face"  # Replace label of entry at index 2
            (label "Facecrush").label   > "Crush face"  # Replace label of first record which has a child with name "label" and value "Facecrush"
            @1                          ^ {...}         # Insert at index 1
            @1                          ~               # Delete entry at index 1
                                        ^ {...}         # This is the first unaddressed entry, so it counts as @0.
                                        ~               # This is the second unaddressed entry, so it counts as @1.
        ]
    }

## Example - Localization

A more realistic example of what localization might look like. This would probably be in a non-English language for a game with its primary language in English, but a developer could choose to put their English language data in patches separate from the gameplay data as well.

    FactionDef-Pirates
    {
        label          > "Pirates"
        description    > "An evil pirate band."
        leaderTitle    > "Boss"

        # Directly replacing the entries in a list. The original data is just a list of strings.
        memberNames
        [
            > "Evil Joe"
            > "Big Nose Lenny"
            > "Machine Gun Martha"
        ]

        # In the original data, each greeting has a text value and a link to a sound file, together in a single table.
        # So we don't replace the entire record. Instead, we replace the text value of each record.
        greetingsDialogueSequence
        [
            text > "I just have one question for you."  # Replace text of first record
            text > "Do you feel lucky?"                 # Replace text of second record
            text > "Well, do you, punk?"                # Replace text of third record
        ]
    }

Note that above, the greetingsDialogueSequence is a list of tables, and each table has a record named `text`. That `text` record is what we're replacing, while we leave the rest of the table untouched since it contains game data.

The below is legal but you don't want to do it in localization! Note that here we replace each entire list record with a new one that has the new `text`. But critically, this will also replace the rest of the record as well, so if there was any game data in there, it's gone now. This is an example of how a translator could freely destroy game data through careless patching.

    greetingsDialogueSequence   # Do not do this! Localization erasing game data.
    [
        > {text "I just have one question for you."}
        > {text "Do you feel lucky?"}
        > {text "Well, do you, punk?"}
    ]

## Possible other ideas

* Mixed data and patch: It would be great if patch and data syntax could be totally unified, so that the data syntax actually means "Add these records in this order and structure", and can be re-used as part of patching as desired. This also automatically supports things like "dotted paths" in data records. E.g if you just want `Goblin{attackDamage 7}` you can just write `Goblin.attackDamage 7'. TOML has a similar feature. This basically would just mean replacing the ^ insert operator with nothing at all.

* Ability to select the nth record with a given name. Otherwise it's quite tricky to address multiple records with the same names, since you need to use index alone.

* A way to load data in "any", "data only", or "patches only" mode. This prevents certain types of mistakes, like translators accidentally defining new content instead of patching existing content.

* A way to enforce localization patches to only be able to modify certain data. This may have to happen outside TyD itself since it depends on context knowledge from the game.

* Make a *patches<> context marker to mark patches. Anything that appears in between the <> angle brackets is parsed as a patch. This is a bit simpler than parsing every record individually, though it makes mixed data and patch impossible.

* Selection by handle attribute:

    (*handle value)     Select a node by its handle
    PawnKind(*handle GoblinBase).speed               > 25                   # Replace a value in a def based on its handle

## Comments

Comments on this proposal are welcome!