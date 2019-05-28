#  Patches proposals

This folder contains proposals for a patch system for TyD. Patches allow data to modify other data that was previously loaded.

These would be for use by:

* Localizations
* Mods
* Expansion packs

Conceptually, a patch can even be though of as a sort of line in a script. Each patch carries out one operation on the data, modifying it in some specific way. Thus order of the patches is critical.

We had two patch-like systems in RimWorld: One for localization, and a separate one for modding. The localization system was relatively clean, but could have been made more concise and easy to use. The modding system worked, but was very verbose - it was based on XML's XPath system. This proposal attempts to do better than both, in one syntax.

Look at the individual proposal files to see how this might work in detail.

## Comments

Comments are welcome, on this proposal and all variants.

## General ideas

* Mixed data and patch: It would be great if patch and data syntax could be totally unified, so that the data syntax actually means "Add these records in this order and structure", and can be re-used as part of patching as desired. This also automatically supports things like "dotted paths" in data records. E.g if you just want `Goblin{attackDamage 7}` you can just write `Goblin.attackDamage 7'. TOML has a similar feature. This basically would just mean replacing the ^ insert operator with nothing at all.

* A way to load data in "any", "data only", or "patches only" mode. This prevents certain types of mistakes, like translators accidentally defining new content instead of patching existing content.

* A way to enforce localization patches to only be able to modify certain data. This may have to happen outside TyD itself since it depends on context knowledge from the game.

* Make a *patches<> context marker to mark patches. Anything that appears in between the <> angle brackets is parsed as a patch. This is a bit simpler than parsing every record individually, though it makes mixed data and patch impossible.
