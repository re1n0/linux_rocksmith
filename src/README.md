# What is this?

This is basically the factory for the guides. I make changes here so I don't have to edit every single file when I want to do a change. It also gives me consistency over files.

This is a rewrite of the original (more complicated) src, which can be found [here](https://github.com/theNizo/linux_rocksmith/tree/11747003673d8f47fd46b355962b8bee2d65f58b/src).

# How does this work?

I have the base structure in base.md. I use variables which have the format of 000-name-000. For each variable, there is a folder containing files named after distros, sound system, or both. I choose the file that fits the current guide.

[generate.sh](/src/generate.sh) creates the guides and puts them in inside [guides/setup/](../guides/setup). Full-line Variable replacements are handled inside [generate.sh](/src/generate.sh), inline changes are made with [replace-inline.sh](/src/replace-inline.sh).

The only more complex part is the [section compiling wineasio](/src/wineasio-install/).
