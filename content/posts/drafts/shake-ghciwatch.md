# shake and ghciwatch

I've found that `ghciwatch` improves on `ghcid` in ways too numerous to mentions, though it does have some issues of its own. In particular, you'll want to [disable debouncing on Linux](https://github.com/MercuryTechnologies/ghciwatch/issues/356) (as seen in all commands below), and [it doesn't support loading multiple Cabal components](https://github.com/MercuryTechnologies/ghciwatch/issues/316) (not likely to be a major issue for Shake scripts).

stitch together GhciWatch and Shake to get reload-on-save working smoothly

note that I may want some specific rules for this site, for bespoke things like adding a new `.md` file

https://stackoverflow.com/questions/55914563/can-shake-generate-dependency-graph-in-graphviz-format
