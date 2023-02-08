---
title: "Getting GitHub links to a project in vim"
date: 2023-02-02T18:50:55-03:00
slug: "vim-github-link"
description: "A little tutorial on how to use vimscript."
draft: false
tags: ["vim", "github"]
math: false
toc: false
---

Lately, at work, I've been writing some documentation that frequently references
part of the code. I generaly use vim to navigate code. Since it's optimized for
my daily workflow, getting to where I need is no work at all. After generating
some links through GitHub, I decided it wouldn't scale well. So I decided I
would try to write a utility function in vimscript that generates a link
anchored in the line my cursor is at. Shouldn't be that hard.

I don't know a whole lot of vimscript. I know just enough to hack a bunch of
snippets I saw here and there and get the functionality I want. If you ever
visit my dotfiles, you'll see the mess my `.vimrc` is. Indeed, for the task I
did the same: a bunch of Google searchs, a few Stack Overflow articles, copy and
paste here and there, and I got where I wanted to. This is not a tutorial on
best practices, but just a simples walkthrough on me trying to get where I
wanted and what I learned through the process.

## Declaring a function

So, the first thing I know is that I wanted to group a bunch of functionality,
and that sounds like a function. This is how you declare a function in
vimscript:

```viml
function! GitHubLink()
    echo "Hello, world!"
endfunction
```

There you have the classic first "Hello, world!". Vimscript functions must start
with a capital letter, that's an important thing. Now, how do you call it? You
can execute commands in the vim command line. Since it's a function, you have to
use the `call` command: `call GitHubLink()`. This should display the message we
echoed.

Another thing that will be important later is arguments. You can have them in
your functions too.

```viml
function! MyFunc(myArg)
    echo a:myArg
endfunction
```

Notice that you need to prefix your parameter's name with `a:` when using it.
This is called a namespace and and is specific for variable arguments. There are
other namespaces, but they won't be used for what we'll do.

So we got the first crucial part going. Now we need to get our data.

## GitHub's link parts

A GitHub link for a file anchored in a line has the following pattern:

`https://github.com/<user>/<repo>/blob/<commit>/<file-path>#L<line-number>`

The commit part is a crucial one. Since I want documentation to always refer to
the part of the code I'm talking about, (Most of them deal with changes and are
meant for historical purposes.) pointing the current commit should is necessary.
Generally, you could simply use `master` to refer to the most up-to-date code
tree.

So, how to get these values? What I come up with:

* The `user` can be retrieved from the remote. We can list them, in a shell,
    with `git remote --verbose`. With some `sed` magic, we can extract only the
    user part;
* The `project` should be the name of the root folder, even though you can
    rename it. I won't bother with it, since I don't tend to rename my
    repository folders. This, of course, is not the only assumption we have: the
    whole goal is to generate GitHub links, even though it's possible to host
    git repositories in other hosts. So it's fine to also assume this for the
    getting the project name; (Another option, though with the same holdbacks,
    is to also use the remote for that.)
* rename it.
* The `commit` hash can be retrieved with `git rev-parse HEAD`;
* `file-path` should be straight-forward to get from the editor itself. We'll
    discuss it bellow;
* The `line-number` also should be possible to be given by the editor.

## Executing shell commands

There's the great plugin [git-fugitive](https://github.com/tpope/vim-fugitive)
that we could use, but, in essence, all it does is deal with some annoying
things one would not want to deal with _after_ it invoked the `git` CLI in a
subshell. So executing shell commands should suffice our needs.

To invoke a shell command we can run `system('command arg1 arg2 ...')`. It will
be executed by the shell configured in your system, (To know which, run `:set
shell?`. Most likely it will be bash.) so you can do all everything you can do
in a shell, including pipes, if statements, variable expansion etc. If you wanna
know more about the `system` function, the rule of vim is to ask for help:
`:help system`. That will get you a long way.

As said, the variables we want to extract are the `user` and `commit`. To get
the commit we would simply run `system('git rev-parse HEAD')`. The user one
is bit trickier and involves the `sed` magic: `system('git remote --verbose |
sed -E "s|.*:(.*)/.*|\1|" | head -n1')`.[^1]

Now, there's a special thing we'll need to do. The output from shell commands
come with a newline (`\n`). This won't be good when we get to the string
interporlation, to assemble our link. To get rid of it we can use the `trim`
function. By default, it removes all whitespaces in the beginning and end of a
string. We just need to wrap any `system` call with a `trim` and we're good.

A last good thing to note: we need to store these values in variables. For that,
we use `let` statements. So, to store a string in a variable we would write:
`let foo = 'bar'`. Pretty simple.

## General vim functions for the rest we need

So, the next thing we need to get is the `project`. We assume it's the name of
the root folder of our project. We can get the absolute path for the root folter
with `getcwd()`. Still, we want only the name of the folder, not the entire
path. There's another function called `fnamemodify`, which can get various parts
of path depending on what we ask for. To get the "tail" of a file name, we can
use the `:t` option. (For all options: `:help filename-modifiers`.) So we would
have something like that, with all wired together: `fnamemodify(getcwd(),
':t')`.

Next thing is the `file`. There's this function called `expand` that will render
_wildcards_. Vim has several wildcard, one of which being `%`, the current file.
Thus, to get our current file: `expand('%')`. This will return the relative path
to our file in relation to the root folder.

The last think is the `line-number`. To get it is pretty simple: `line('.')`.
The `.` says we want the line position of where our cursor is at. (For more
expressions: `:help line`.)

## Assembling a link

With all the needed variables, we can, now, join them together to form a
complete link. String concatenation, in vimscript, is done with the `.`
operator:

```viml
let link = 'https://github.com/' . user . '/' . project . '/blob/' . commit . '/' . file . '#L' . linenumber
```

## Copying the result to the clipboard

We learned almost everything. Now there's this detail which is getting the link
to our clipboard. There are some differences between MacOS and Linux when it
comes to interacting with the system's clipboard. Vim uses these things called
registers to interface with it. Registers are general buffers that store text.
They are used for different purposes. There are two special buffers related to
the clipboard: `"*` and `"+`. (All registers are accessed with a `"` in normal
mode.) MacOS's clipboard is wired to `"*`, while Linux's (In reality, depends on
your graphical setup, but most people use Xorg.) is wired to `"+`.

It's possible to write directly to a register as you would to with a variable.
Register variables are accessed with a `@` instead of a `"`. Since we want to
switch according to our current OS, we'll use an `if` statement.

```viml
function! ToClipboard(string)
    if has('mac') | let @* = a:string | else | let @+ = a:string | endif
endfunction
```

Notice that I used `|` to join everything into a single line. If you wish to
write the statement with line breaks, just substitute these with line breaks. I
prefer to write short statements in one line. I think it makes it more readable.

## What I got

Having gathered all the needed parts, this is what I ended up with:

```viml
function! ToClipboard(string)
    if has('mac') | let @* = a:string | else | let @+ = a:string | endif
endfunction

function! GitHubLink()
    let project = fnamemodify(getcwd(), ':t')
    let user = trim(system('git remote --verbose | sed -E "s|.*:(.*)/.*|\1|" | head -n1'))
    let commit = trim(system('git rev-parse HEAD'))
    let file = expand('%')
    let linenumber = line('.')
    return 'https://github.com/' . user . '/' . project . '/blob/' . commit . '/' . file . '#L' . linenumber
endfunction
```

The last thing to do is wire it to some key. Who wants to call the function
manually everytime one wants to use it? I use vim to have everything a few
key strokes away. This should be straight-forward for every vim user, but this
is how you wire up our newly written function with a key sequence:

```viml
nnoremap <silent> <C-G>l <Cmd>call ToClipboard(GitHubLink())<CR>
```

Every time we press `Ctrl+G+l` in normal mode vim should invoke our new function
and copy the generated link to our clipboard. No more searching through GitHub's
web interface for me. A step closer to living entirely in the terminal.

[^1]: If you want to learn some `sed`, I'd recommend the [tutorial written by
    Bruce Barnnet](https://www.grymoire.com/Unix/Sed.html).
