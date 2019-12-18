---
layout: post
title: "Use `credo` Linter in Git `pre-commit` Hook"
description: "copy-paste solution to start using credo linter in git pre-commit hook"
category: hacking
tags:
  - tricks
  - elixir
---

[`credo`](http://credo-ci.org/) is [great](https://github.com/rrrene/credo).

Every single Elixir project all around should use it. Besides fancy `mix` task
to report project styling issues, it has bindings to most of modern editors,
like [`neovim`](https://www.dailydrip.com/topics/elixirsips/drips/neovim-for-elixir),
[`emacs` through alchemist.el](https://github.com/tonini/alchemist.el) and even [`Atom`](https://atom.io/packages/linter-elixir-credo).

The greatest thing about `credo` is it provides the _meaningful shell return codes_,
making it quite easy to plug it as a linter into `git` hooks toolchain. Let’s do it:

### Include `credo` dependency into `mix.exs` project file

```elixir
  defp deps do
    [
      ...
      {:credo, "~> 0.7", only: [:dev, :test]},
      ...
    ]
  end
```

if you are reading this article from the future, immediately run

```
$ mix deps.update credo
```

to get the up-to-date version. It’s safe, since we’d use it in non-prod
environments only. Let’s try it in action.

### Running `credo` for the first time

```
$ mix credo

Checking 36 source files ...

  Software Design
┃
┃ [D] ↗ Found a FIXME tag in a comment: # FIXME FIXME FIXME
┃       lib/my_app/file1.ex:194 (MyApp.File1.fun1)
┃ [D] → Found a TODO tag in a comment: # Todo: should round here?
┃       lib/my_app/file2.ex:53 (MyApp.File2.fun2)

  Code Readability                                                              
┃
┃ [R] → Modules should have a @moduledoc tag.
┃       lib/my_app/file1.ex:4:13 (MyApp.File1)
┃ [R] → Modules should have a @moduledoc tag.
┃       lib/my_app/file3.ex:3:11 (MyApp.File3)

  Refactoring opportunities                                                     
┃
┃ [F] → Function body is nested too deep (max depth is 2, was 4).
┃       lib/my_app/file1.ex:179:21 (MyApp.File1.fun1)

  Warnings - please take a look                                                 
┃
┃ [W] ↗ Prefer lazy Logger calls.
┃       lib/my_app/file1.ex:91 (MyApp.File1.fun3)
┃ [W] ↗ Prefer lazy Logger calls.
┃       lib/my_app/file1.ex:82 (MyApp.File1.fun2)
┃ [W] ↗ Prefer lazy Logger calls.
┃       lib/my_app/file1.ex:169 (MyApp.File1.fun3)

Please report incorrect results: https://github.com/rrrene/credo/issues

Analysis took 1.1 seconds (0.02s to load, 1.1s running checks)
220 mods/funs, found 3 warnings,
                     1 refactoring opportunity,
                     2 code readability issues,
                     2 software design suggestions.
```

Here we go. First run report is always depressive and driving me bonkers.
I was writing this code today morning, it’s, you know, kinda perfect. I hope
you’ll get zero warnings and at most one `TODO` tag found, but I never get there.
First run turns me into a couple of hours of refactoring. The good thing is
from now on it’s fairly easy to keep the project in clean state, that
satisfies even such a captious judge as `credo`.

### Configure `credo`

The very straightforward way to make everything tuned would be to:

```
cd config
# wget https://raw.githubusercontent.com/rrrene/credo/master/.credo.exs
mix credo gen.config
vim .credo.exs
cd -
```

The file is self-documented. Just go tweak it.

### Making it happen recurrently

Well, I knew one guy who had put running linters into `crontab` at 2AM.
He started every morning with few cups of coffee and several hours of
refactoring. I am not as brave. I use git hooks to lint my code (save for editor
plugins.)

Let’s configure our `.git/config` in the first place. Just add
the following section to it:

```
[credo]
  terminate = 16
```

The above will terminate the commit when there are warnings. Check [how `credo`
calculates an exit status](https://github.com/rrrene/credo#exit-status) for details.

OK, now we are to create a hook itself. You might just put this in your
`.git/hooks/pre-commit` local hook (create this file if it’s not existing yet):

```sh
#!/bin/sh
#
# Linter Elixir files using Credo.
# Called by "git receive-pack" with arguments: refname sha1-old sha1-new
#
# Config
# ------
# credo.terminate
#   The credo exit status level to be considered as “not passed”—to prevent
#   git commit until fixed.

# Config
terminate_on=$(git config --int credo.terminate)
if [[ -z "$terminate_on" ]]; then terminate_on=16; fi

# test it :: run tests before commit (silently)
mix test 2>&1 >/dev/null
TEST_RES=$?
if [ $TEST_RES -ne 0 ]
then
  echo ""
  echo "☆ ==================================== ☆"
	echo "☆  Some tests are failed.              ☆" >&2
	echo "☆  Please fix them before committing.  ☆" >&2
  echo "☆ ==================================== ☆"
  echo ""
  exit $TEST_RES
fi
echo ""
echo "★ ============================== ★"
echo "★   Tests passed successfully.   ★"
echo "★ ============================== ★"
echo ""

# lint it :: credo checks before commit
mix credo
CREDO_RES=$?
if [ $CREDO_RES -ge $(terminate_on) ]; then
  echo ""
  echo "☆ ============================================= ☆"
	echo "☆ Credo found critical problems with your code  ☆" >&2
	echo "☆   and commit can not proceed. Please examine  ☆" >&2
	echo "☆   log above and fix issues before committing. ☆" >&2
  echo "☆ ============================================= ☆"
  echo ""
  exit $CREDO_RES
fi
if [ $CREDO_RES -le 9 ]; then CREDO_RES=" $CREDO_RES"; fi
echo ""
echo "★ ============================== ★"
echo "★   Credo passed successfully.   ★"
echo "★   Exit value is: (total: $CREDO_RES).  ★"
echo "★ ============================== ★"
echo ""

# Finished
exit 0
```

Before `credo` we also run tests. If you are like me and want to commit
code even while all the tests are failing (hopefully providing the comment
like “:bomb: [BROKEN] Evening commit! Do not use at home or school!”,)
just supply `--no-verify` option to `git commit`.

And yes, we are done. Try to `git commit` and you’ll see as your code will
be tested and credo’ed for you automagically.
