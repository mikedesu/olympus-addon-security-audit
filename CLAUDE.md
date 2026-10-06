# CLAUDE.md

## olympus-addon-security-audit

As a security-minded fellow, I traditionally do not trust mods or add-ons in games, and have always been hesitant about it to a point of generally never modding a game, with few exceptions.

However, as I have joined Olympus CXVII in WoW Forever, I am considering checking out the add-on and seeing if it improves my guild experience.

Before I do so, I want to comb over the codebase of olympus-addon and look for things that might be abusive or harmful or cause-for-concern to someone who installs it.

The git repo is cloned locally at /home/darkmage/src/olympus-addon/ and all work related to the audit will go in this local git repo.

The addon is primarily composed of lua code, so the real question becomes about what lua interpreter wow forever classic uses.

I honestly don't anticipate any malicious code present in the codebase. 

I want you to write a README.md first, and then you will break the audit up into stages that can be managed in a checkbox-list like so:

```
[x] completed item 0
[ ] item 1
[ ] item 2
[ ] item 3
```

This will go into PLAN.md

Then, you will work through each item in PLAN.md, logging findings and summaries into session-NNN.md where NNN is the current session number.

This way, I can have confidence that the plugin doesn't do anything malicious to my machine.

