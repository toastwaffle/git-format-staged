# git-format-staged

[![Build Status](https://travis-ci.org/hallettj/git-format-staged.svg?branch=master)](https://travis-ci.org/hallettj/git-format-staged)

Consider a project where you want all code formatted consistently. So you use
a formatting command. (For example I use [prettier-standard][] in my
Javascript projects.) You want to make sure that everyone working on the
project runs the formatter, so you use a tool like [husky][] to install a git
pre-commit hook. The naive way to write that hook would be to:

- get a list of staged files
- run the formatter on those files
- run `git add` to stage the results of formatting

The problem with that solution is it forces you to commit entire files. At
worst this will lead to contributors to unwittingly committing changes. At
best it disrupts workflow for contributors who use `git add -p`.

git-format-staged tackles this problem by running the formatter on the staged
version of the file. Staging changes to a file actually produces a new file
that exists in the git object database. git-format-staged uses some git
plumbing commands to send content from that file to your formatter. The command
replaces file content in the git index. The process bypasses the working tree,
so any unstaged changes are ignored by the formatter, and remain unstaged.

After formatting a staged file git-format-staged computes a patch which it
attempts to apply to the working tree file to keep the working tree in sync
with staged changes. If patching fails you will see a warning message. The
version of the file that is committed will be formatted properly - the warning
just means that working tree copy of the file has been left unformatted. The
patch step can be disabled with the `--no-update-working-tree` option.

[prettier-standard]: https://www.npmjs.com/package/prettier-standard
[husky]: https://www.npmjs.com/package/husky


## How to install

Requires Python version 3 or 2.7.

Install as a development dependency in a project that uses npm packages:

    $ npm install --save-dev git-format-staged

Or install globally:

    $ npm install --global git-format-staged

If you do not use npm you can copy the
[`git-format-staged`](./git-format-staged) script from this repository and
place it in your executable path. The script is MIT-licensed - so you can check
the script into version control in your own open source project if you wish.


## How to use

For detailed information run:

    $ git-format-staged --help

The command expects a shell command to run a formatter, and one or more file
patterns to identify which files should be formatted. For example:

    $ git-format-staged --formatter 'prettier --stdin' 'src/*.js'

That will format all files under `src/` and its subdirectories using
`prettier`. The file pattern is tested against staged files using Python's
[`fnmatch`][] function: each `*` will match nested directories in addition to
file names.

[`fnmatch`]: https://docs.python.org/3/library/fnmatch.html#fnmatch.fnmatch

The formatter command must read file content from `stdin`, and output formatted
content to `stdout`.

Files can be excluded by prefixing a pattern with `!`. For example:

    $ git-format-staged --formatter 'prettier --stdin' '*.js' '!flow-typed/*'

Patterns are evaluated from left-to-right: if a file matches multiple patterns
the right-most pattern determines whether the file is included or excluded.

git-format-staged never operates on files that are excluded from version
control. So it is not necessary to explicitly exclude stuff like
`node_modules/`.

### Check staged changes with a linter without formatting

Perhaps you do not want to reformat files automatically; but you do want to
prevent files from being committed if they do not conform to style rules. You
can use git-format-staged with the `--no-write` option, and supply a lint
command instead of a format command. Here is an example using ESLint:

    $ git-format-staged --no-write -f 'eslint --stdin >&2' 'src/*.js'

If this command is run in a pre-commit hook, and the lint command fails the
commit will be aborted and error messages will be displayed. The lint command
must read file content via `stdin`. Anything that the lint command outputs to
`stdout` will be ignored. In the example above `eslint` is given the `--stdin`
option to tell it to read content from `stdin` instead of reading files from
disk, and messages from `eslint` are redirected to `stderr` (using the `>&2`
notation) so that you can see them.

### Set up a pre-commit hook with Husky

Follow these steps to automatically format all Javascript files on commit in
a project that uses npm.

Install git-format-staged, husky, and a formatter (I use prettier-standard):

    $ npm install --save-dev git-format-staged husky prettier-standard

Add a `"precommit"` script in `package.json`:

    "scripts": {
      "precommit": "git-format-staged -f prettier-standard '*.js'"
    }

Once again note that the `'*.js'` pattern is quoted! If the formatter command
included arguments it would also need to be quoted.

That's it! Whenever a file is changed as a result of formatting on commit you
will see a message in the output from `git commit`.


## Comparisons to similar utilities

There are other tools that will format or lint staged files. What distinguishes
git-format-staged is that when a file has both staged and unstaged changes
git-format-staged ignores the unstaged changes; and it leaves unstaged changes
unstaged when applying formatting.

Some linters (such as [precise-commits][]) have an option to restrict linting
to certain lines or character ranges in files, which is one way to ignore
unstaged changes while linting. The author is not aware of a utility other than
git-format-staged that can apply any arbitrary linter so that it ignores
unstaged changes.

Some other formatting utilities (such as [pre-commit][])
use a different strategy to keep unstaged changes unstaged:

1. stash unstaged changes
2. apply the formatter to working tree files
3. stage any resulting changes
4. reapply stashed changes to the working tree.

The problem is that you may get a conflict where stashed changes cannot be
automatically merged after formatting has been applied. In those cases the user
has to do some manual fixing to retrieve unstaged changes. As far as the author
is aware git-format-staged is the only utility that applies a formatter without
touching working tree files, and then merges formatting changes to the working
tree. The advantage of merging formatting changes into unstaged changes (as
opposed to merging unstaged changes into formatting changes) is that
git-format-staged is non-lossy: if there are conflicts between unstaged changes
and formatting the unstaged changes win, and are kept in the working tree,
while staged/committed files are formatted properly.

Another advantage of git-format-staged is that it has no dependencies beyond
Python and git, and can be dropped into any programming language ecosystem.

Some more comparisons:

- [lint-staged][] lints and formats staged files. At the time of this writing
  it does not have an official strategy for ignoring unstaged changes when
  linting, or for keeping unstaged changes unstaged when formatting. But
  lint-staged does provide powerful configuration options around which files
  should be linted or formatted, what should happen before and after linting,
  and so on.
- the one-liner
  `git diff --diff-filter=d --cached | grep '^[+-]' | grep -Ev '^(--- a/|\+\+\+ b/)' | LINT_COMMAND`
  (described [here][lint changed hunks]) extracts changed hunks and feeds them
  to a linter. But linting will fail if the linter requires the entire file for
  context. For example a linter might report errors if it cannot see import
  lines.

[precise-commits]: https://github.com/nrwl/precise-commits
[pre-commit]: https://pre-commit.com/#pre-commit-during-commits
[lint-staged]: https://github.com/okonet/lint-staged
[lint changed hunks]: https://github.com/okonet/lint-staged/issues/62#issuecomment-383217916
