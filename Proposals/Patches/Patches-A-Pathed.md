# Patches proposal A - Pathed patches

## Basics

The format for each patch record is:

    PATH_TO_NODE OPERATOR (VALUE)

The parser knows which records are patches based on their syntax.

## Node paths

A node path is a sequence of actions, each of which moves the selection to a new node in some way relative to the currently selected node.

There is always exactly one node selected. Initially, the selected node is the "invisible root", which is the anonymous, implied parent of all the root nodes.

The path actions are:

* `/CHILDNAME`: Select the first child of the current node with a given name.
* `@INDEX`: Select the child of the current node at index `INDEX`. If `INDEX` is negative, it counts backwards from the last child node; -1 is the last list element. `-0` is accepted as an index after the last child node and can be used for insertions operations. This works for both lists and tables.
* `(GRANDCHILDNAME "VALUE")`: Select the first child of the current node which itself has a string child with the given name and value. Quotation marks are required around the value.

## Operations

After you select a node using a path, you want to change it somehow. These define the operations that can change the selected node.

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

## Discussion

You can freely mix patches and declarations in the same file.

The parser can distinguish patches from data because patch paths must start with a pathaction, and all pathactions start with characters that are illegal as TyD node names. So, the TyD parser can easily know, from the first character, when it's reading a patch instead of data. This'll make parsing faster.

## Ideas

* Add an optional `?` character at the end of the path, to mean 'optional'. If the node isn't found, an optional patch will be silently ignored, whereas a standard patch would generate an error.

* Ability to select the nth record with a given name. Otherwise it's quite tricky to address multiple records with the same names, since you need to use index alone.

* Custom or exotic operator syntax. We use the below syntax to allow defining either custom patch operator, or more exotic patch operators. So patches can do weird game-specific things like, say, capitalize all text, offset numbers by a certain amount, and so on.

    &OPERATOR_NAME OPERATOR_DATA

* Some way to select and affect multiple nodes at once.

* Selection by handle attribute:

    (*handle value)     Select a node by its handle
    PawnKind(*handle GoblinBase).speed > 25    # Replace a value in a def based on its handle

* Allow un-quoted strings in the `(NAME VALUE)` path action.