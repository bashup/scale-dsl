# Scala-inspired Embedded DSLs for Bash

Bash, by its nature, is a very "flat" language, that lacks the ability to cleanly define nested data structures  (e.g. JSON, HTML, or any other structured content or configuration).

Enter [scale-dsl](scale-dsl): a bash 3.2+ micro-library that lets you create **S**tructured **C**onfiguration **A**nd **L**anguage **E**xtensions.  By sourcing it, or copy-pasting it into your script, and then adding a few simple handler functions, it lets you write bash "outlines": nested data structures and commands, whose semantics are defined by bash functions.

For example, using the handler functions in [demos/scale-json](demos/scale-json), this bash function:

```shell
contact-as-json() {
  ~ json map; {{
    - first_name str "$1"
    - last_name  str "$2"
    - email      str "$3"
    + tags list; {{
        local tag
        for tag in "${@:4}"; do
            - str "$tag"
        done
    }}
  }}
}
```

can be run to generate valid JSON for arbitrary arguments:

~~~sh
    $ contact-as-json Bob Dobbs the.bob@example.com fake demo
    $ echo "$REPLY"
    {"first_name":"Bob", "last_name":"Dobbs", "email":"the.bob@example.com", "tags":["fake", "demo"]}
~~~

(The above code is pretty fast, too: generating "Bob Dobbs"'s JSON takes about 1/800th of a second on a modest Linux server: less time than it would take to even *start* an external command!)

Of couse, you're not limited to producing JSON, or even text.  Your outline items (and block-accepting functions) can set variables, run programs, or do whatever other kind of computation is required.  It's entirely up to you (and your outline handlers).

SCALE works by adding just three new syntactic structures to bash:

* `~` *shell-command(s)...* `; {{` *block...* `}}`

  Turns *block* into a function that one of the *shell-commands* can call (or not), optionally setting a handler for nested outline items.

* `-` *outline-item-args...*

  Sends *outline-item-args* as arguments to the current outline handler, to be interpreted as data or as a directive/subcommand.

* `+` *outline-item-args...*  `; {{` *block...* `}}`

  Sends *outline-item-args* as arguments to the current outline handler, which can call *block* (or not), optionally changing the handler for nested outline items.  (The `+` is literally just shorthand for `~ -`: i.e., wrapping an outline item as a block-taking command.)

So in the example code above, `json map` is invoking the shell function `json` with the single argument `map`, and then that function invokes the outermost block with an outline handler that processes outline items as JSON object entries (a key, a JSON type, and an optional value).  The `map` and `list` data types process nested blocks in this fashion, while other data types' values can be specified without a nested block.  (Also notice that `-` and `+` are simple bash commands and as such can be enclosed in loops or conditionals, use variable interpolation, etc.)

## Basic Use

The basic idea of SCALE outlines is that a `~`-prefixed command series controls when (or whether) to run the block that follows it, optionally setting an "outline handler" that will be called whenever a  `+` or `-` outline item is executed within that block.  For example, here's the simplest possible outline handler you can use with SCALE:

~~~sh
    $ source scale-dsl  # load the library
    $ ~ ::block echo; {{ - Hello; - World; }}
    Hello
    World
~~~

In this example, the command `::block echo` tells SCALE to run the block and pass any `-` or `+` outline items to `echo` (our outline handler) as arguments.  So `Hello` and `World` both get echoed, as each `-` is expanded at runtime to `echo`.

Now let's try a slightly more complex example, with nesting, and passing prefix arguments to outline handler functions:

~~~sh
    # Set the initial indent
    $ indented() { ::block indent-with ""; }

    $ indent-with() {
    >     # Output the indent and the input
    >     local indent=$1; shift; echo "$indent$*"
    >     # Got a block?  Run it with a deeper indent
    >     ::block indent-with "    $indent"
    > }

    $ ~ indented; {{
    >   - This is a test
    >   + of indenting ; {{
    >     - nested
    >     + deeper ; {{
    >         for i in 1 2 3; do
    >             - yeah "($i)"
    >         done
    >     }}
    >     - back out
    >   }}
    >   - and again
    > }}
    This is a test
    of indenting
        nested
        deeper
            yeah (1)
            yeah (2)
            yeah (3)
        back out
    and again
~~~

Each basic command in the block that begins with `-` or `+` is forwarded to the outline handler (`indent-with ""`) as command line arguments.  So the line `- This is a test` becomes `indent-with "" This is a test`, and `+ of indenting` becomes `indent-with "" of indenting`.  The `indent-with` function then runs any nested block with a deeper indent.

(As you can see, `::block` accepts any number of arguments, not just a command or function name.  This makes it easier to create reusable handler functions, as the extra arguments are included in each expansion of `+` and `-`.)

## Scripting and Composition

Notice that SCALE blocks are still plain bash code, and can therefore use control structures, expressions, parameter expansion, and all other normal commands or functions, in addition to the ones prefixed by  `+`, `-`, or `~`.  Outline handlers are inherited even across function calls, so you can put parts of a data structure into functions and then reuse them from different enclosing structures and even different handlers, e.g.:

~~~sh
    $ say-hello() { - "Hello, $1!"; }

    $ ~ ::block echo; {{ say-hello world; }}
    Hello, world!

    $ ~ indented; {{
    >   + "My response:"; {{ say-hello yourself; }};
    > }}
    My response:
        Hello, yourself!
~~~

Notice how the `-` in the `say-hello` function produces different results in each use, expanding to `echo` or `indent-with "    "`, according to where it's called from.  (Notice, too, that we invoked it as a *normal bash function*, not as a `-` line unto itself.  If we had put a `-` in front of it, we would have ended up calling  `echo say-hello world` and `indent-with "    " say-hello yourself`, which would have produced rather different results!)

Although you *can* separate what's inside a  `{{`...`}}` block into different functions, do note that you *can't* separate a block from its opening `+` or `~`: they **must** be immediately adjacent and within the same enclosing bash block or control structure.  (Otherwise, a bash syntax error will occur.)

(Note, too that a `+` or `~` and its block are invisibly wrapped togethr as a bash compound statement; so e.g. any redirection applied after the closing `}}` will apply to the execution of the commands or outline item preceding the `{{` part.)

### Variables and Parameters

Blocks within a function can use that function's received arguments normally (`$1`, `$2`, etc.), regardless of how deep the block nesting.  Unlike normal bash blocks, however, SCALE blocks can actually have their own local variables, shadowing the ones defined by the function.

Within any given block, local variables from any enclosing blocks or containing/calling functions are also accessible, alternating in precedence with the locals of the outline handler(s) or top-level command that invoked each block.  So, within some nested block, the order of variable lookup or assignment would be:

* `$1, $2...` etc. from the enclosing function or script (no matter how deeply nested the blocks)
* locals defined within that block
* locals defined by the handler that called `::block` to run the block
* locals defined by the enclosing block
* locals defined by the handler that called `::block` to run the enclosing block
* ...
* locals defined by the command that called `::block` on the outermost block
* locals defined outside the outermost block (e.g. the enclosing function, if any)
* locals defined by the caller(s)
* globals

This order applies to the handlers, too, in that they can see local variables defined below themselves in this list.  (But their positional parameters are always their own: they see their own arguments in `$1`, `$2`, etc., not those of the blocks' enclosing function.)

### Controlling Block Execution and Interpretation

If you're designing a DSL or API, note that calling `::block` from a `~` command or `+` item is strictly optional; if you *don't* call it, the code within `{{`...`}}` simply won't be executed.  A given block can only be executed *once*, however: calling `::block` more than once for the same block will have no effect (unless the calls are done inside a subshell).  Calling `::block` with no arguments runs the block without changing the current outline handler.

If no outline handler is defined (i.e. if you use `+` or `-` without an enclosing block), a handler called `::no-dsl` will be invoked (if it exists).  If your program does not have a `::no-dsl` function and a `::no-dsl` command does not exist on your `PATH`, an error will occur.

## Compatibility Notes

SCALE is implemented mainly using bash aliases (for `+` , `-`, `~`, `{{` and `}}`), and the public `::block` function.  Internally, however, SCALE reserves three additional function names for its own use (aside from `::no-dsl`), and has several private variables.  To avoid unwanted and unpredictable behavior, you should not use or define functions named `::^`, `::`, or `^::`, nor should you set, unset, or declare variables named `__blk__`, `__bsp__`, or `__dsl__`.

For optimum performance, SCALE is carefully written to avoid forking.  However, if a block-taking function or outline handler uses blocks itself (or calls other code that does) *before* the enclosing block is executed, a `$(declare -f ::)` substitution is required to save the source code of the not-yet-executed block.  You can avoid this overhead by ensuring that any other block-using code is run *after* your code calls `::block`.

SCALE syntax is mostly compatible with [shellcheck](https://www.shellcheck.net/), except that you need to disable [SC2215](https://github.com/koalaman/shellcheck/wiki/SC2215) (Commands beginning with `-`) and [SC1054](https://github.com/koalaman/shellcheck/wiki/SC1054) (doubled braces).  Adding `# shellcheck disable=1054,2215` on the line before a block begins (or the enclosing function, if any) will disable them locally, or you can disable them at the project level if you prefer.

Finally, note that if you need to run actual programs or functions named `+` , `-`, `~`, `{{` or `}}`, you can simply prefix them with a `\` or enclose them in quotes.  This will stop the shell from expanding them as aliases.  If the code can't easily be changed, you can simply source it *before* `scale-dsl`, or else disable the `expand_aliases` shell option while sourcing it, and re-enable it afterward.  Bash expands aliases while code is being *compiled*, not when it's run, so you can have DSL-using and non-DSL-using functions in the same program, with different understandings of what `-`, `+`, etc. mean.

(Also, if you're using bash 4.4's new `local -` construct, please note that it has [a bug that disables alias expansion in scripts](https://lists.gnu.org/archive/html/bug-bash/2018-12/msg00023.html).  If you call such a function before all of your source code has been read, you'll need to re-enable alias expansion, *outside* the function, after the call is made.)

## License

<p xmlns:dct="http://purl.org/dc/terms/" xmlns:vcard="http://www.w3.org/2001/vcard-rdf/3.0#">
  <a rel="license" href="http://creativecommons.org/publicdomain/zero/1.0/"
  ><img src="https://licensebuttons.net/p/zero/1.0/80x15.png" style="border-style: none;"
        alt="CC0" /></a><br />
To the extent possible under law, <a rel="dct:publisher" href="https://github.com/pjeby"
><span property="dct:title">PJ Eby</span></a> has waived all copyright and related or
    neighboring rights to <span property="dct:title">bashup/scale-dsl</span>.  This work is
    published from: <span property="vcard:Country" datatype="dct:ISO3166" content="US"
                          about="https://github.com/bashup/scale-dsl">United States</span>.
</p>
