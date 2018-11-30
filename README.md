# Scala-like Embedded DSLs for Bash

Bash, by its nature, is a very "flat" language, that lacks the ability to cleanly define nested data structures  (e.g. JSON, HTML, or any other structured content or configuration).

Enter [scale-dsl](scale-dsl): a bash 3.2+ micro-library that lets you create **S**tructured **C**onfiguration **A**nd **L**anguage **E**xtensions.  By sourcing it, or copy-pasting it in your script, you get access to these three simple syntax enhancements:

* `---` *block-command...* `&& {{` *block...* `}}`
* `+` *block-phrase...*  `&& {{` *block...* `}}`
* `-` *item-phrase...*

Block commands and phrases wrap the execution of their attached blocks, and control how any nested phrases will be interpreted.  With properly implemented command functions, you can then write code like this:

```shell
json-for-contact() {
    --- json map: && {{
        - first_name str "$1"
        - last_name  str "$2"
        - email      str "$3"
        + tags list: && {{
            for tag in "${@:4}"; do
                - str "$tag"
            done
        }}
    }}
}
```

or this:

```shell
hello-world-html() {
    --- html && {{
        + head && {{ - title : "Hello world"; }}
        + body && {{
            + article'#content' && {{
                - h1.plain id=top-heading : "Welcome!"
                - p : "Hello, $1"
            }}
        }}
    }}
}
```

...with all the nesting and escaping handled properly.

The basic idea here is that the `---`-prefixed command controls when or whether to run the block that immediately follows it, optionally setting a "DSL interpreter" that will be called whenever a  `+` or `-` phrase is executed from within that block.  For example, here's the simplest possible DSL you can make with SCALE:

~~~sh
    $ source scale-dsl  # load the library
    $ --- dsl: echo && {{
    >     - Hello
    >     - World
    > }}
    - Hello
    - World
~~~

In this example, the command `dsl: echo` tells SCALE to run the block and pass any `-` or `+` phrases to `echo` as arguments.  So `- Hello` and `- World` get echoed.

Now let's try a slightly more complex example, with nesting:

~~~sh
    # Set the initial indent
    $ indented() { dsl: indent-with ""; }

    $ indent-with() {
    >     # Output the indent and the input, skipping the + or - in between
    >     local indent=$1 block=$2; shift 2; echo "$indent$*"
    >     # Got a block?  Run it with a deeper indent
    >     if [[ $block == "+" ]]; then dsl: indent-with "    $indent"; fi
    > }

    $ --- indented && {{
    >     - This is a test
    >     + of indenting && {{
    >         - nested
    >         + deeper && {{
    >            for i in 1 2 3; do
    >                - yeah
    >            done
    >         }}
    >         - back out
    >     }}
    >     - and again
    > }}
    This is a test
    of indenting
        nested
        deeper
            yeah
            yeah
            yeah
        back out
    and again
~~~

Each basic command in the block that begins with `-` or `+` is forwarded to the block handler as command line arguments.  So the line `- This is a test` becomes `indent-with "" - This is a test`, and `+ of indenting` becomes `indent-with "" + of indenting`.  The `indent-with` function then detects the presence of a block via the `+`, and runs the nested block with a deeper indent.

## Scripting and Composition

Notice that DSL blocks are still plain bash code, and can therefore use control structures, expressions, parameter expansion, and all other normal commands or functions, in addition to the ones prefixed by  `+`, `-`, or `---`.  Block handlers are dynamically carried down into function calls, so you can put parts of a data structure into functions and then reuse them from different enclosing structures, e.g.:

~~~sh
    $ hello-func() { - "Hello, $1!"; }

    $ --- dsl: echo && {{ hello-func world; }}
    - Hello, world!

    $ --- indented && {{
    >     + "My response:" && {{ hello-func yourself; }};
    > }}
    My response:
        Hello, yourself!
~~~

Notice how the `-` in the `hello` function produces different results in each use, expanding to `echo -` or `indent-with "    " -`, according to where it's called from.  (Notice, too, that we called it as a *normal bash function*, not as a `-` line unto itself.  If we had, then we would have ended up calling e.g. `echo - hello-func world` or `indent-with "    " - hello-func yourself`, which would have produced rather different results!)

Although you can separate what's inside a  `{{`...`}}` block or its header into different functions, do note that you *can't* separate the block itself from its opening `+` or `---`: they **must** be immediately adjacent and within the same enclosing bash block or control structure.  (Otherwise, a syntax error will occur.)

### Variables and Parameters

Blocks within a function can use that function's arguments (`$1`, `$2`, etc.) normally, regardless of how deep the nesting.  Local variables from the enclosing blocks are also accessible, alternating in precedence with the locals of the handlers.

So, within some nested block, the order of variable lookup would be:

* `$1, $2...` etc. from the enclosing function
* locals defined within that block
* locals defined by the code that called `dsl:` to run it
* locals defined by any enclosing block
* locals defined by the code that called `dsl:`
* ...
* locals defined by the outermost block
* locals defined outside the outermost block

This order applies to the handlers, too, in that they can see local variables from further down this list.  (But their positional parameters are their own; they don't see the enclosing function's arguments.)

### Controlling Block Execution and Interpretation

If you're designing a DSL or API, note that calling `dsl:` from a `---` command or `+` phrase is strictly optional; if you *don't* call it, the code within `{{`...`}}` simply won't be executed.  A given block can also only be executed *once*: calling `dsl:` more than once will have no effect (unless the calls are done inside a subshell).  Calling `dsl:` with no arguments runs the block without changing the current interpreter.

If no block handler is defined (i.e. if you use `+` or `-` without an enclosing block), a block handler called `::no-dsl` will be invoked, if it is defined.  (If your program does not have a `::no-dsl` function and a `::no-dsl` command does not exist on your `PATH`, an error will occur.)

## Compatibility Notes

SCALE is implemented mainly using bash aliases (for `+` , `-`, `---`, `{{` and `}}`), and the public `dsl:` function.  Internally, however, SCALE reserves three additional function names for its own use (aside from `::no-dsl`), and has several private variables.  To avoid unwanted and unpredictable behavior, you should not use or define functions named `::__`, `::`, or `__::`, nor should you set, unset, or declare variables named `__blk__`, `__blarg__`, `__bstk__`,  `__bptr__`, or `__dsl__`.

For optimum performance, SCALE is carefully written to avoid forking.  However, if a header function or handler uses blocks itself (or calls other code that does) *before* the enclosing block is executed, a `$(declare -f ::)` substitution is required to save the source code of not-yet-executed block.  You can avoid this overhead by ensuring that any other block-using code is run *after* your handler calls `dsl:`.

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
