<!--
Stub only — do not put content here.

The real text lives in the binary-codegraph subrepo (external-repo/binary-codegraph/CLAUDE.md)
so that it travels with that repository when the work is migrated to another machine.

Why the indirection: Claude Code loads CLAUDE.md files in SUBDIRECTORIES of the working
directory lazily (only once Claude reads a file in that subtree), and does NOT re-inject them
after /compact. An @-import inside the launch-root CLAUDE.md is expanded into context at
session start and is re-injected with this file after compaction, so the convention stays
loaded through long sessions.

Session cwd is G:/codingnet/ghidra. binary-codegraph is a SEPARATE git repository, excluded
from this one via .git/info/exclude; nothing here is read by its gates.

This file itself does not travel — recreate it (one import line) on a new machine.
-->

@external-repo/binary-codegraph/CLAUDE.md
