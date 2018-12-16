## Block Nesting and Deferred commands

It's possible in the header of a block (the part before the `{{`) to call a function that defines another function that uses a block.  To handle this scenario, `scale-dsl` implements a "block stack" using the `__bstk__` array and` __bsp__` stack pointer.  Without it, a scenario like this one would fail:

~~~sh
    $ set -u  # Show errors if any undefined variables are used
    $ . scale-dsl

    $ f1() { ~ dsl: && {{ echo "nested"; }}; dsl:; }
    $ ~ f1 && {{ echo "not nested"; }}
    nested
    not nested
~~~

Given that the block stack exists, it's possible to tweak its contents to create a golang-style "defer" operation, e.g.:

~~~sh
    $ defer:(){
    >     ((__bsp__&&$#))||return
    >     local r;printf -v r \ %q "$@";__bstk__[__bsp__-1]="$r;${__bstk__[__bsp__-1]}"
    > }

    $ defer: echo 42   # fail, not in a block
    [1]

    $ ~ defer:; {{ :; }}   # fail, no arguments
    [1]

    $ ~
    >   echo; echo "begin header"
    >   defer: echo "defer 1 from header"
    >   dsl:
    >   defer: echo "defer 2 from header"
    >   echo "end header"; echo
    >   {{
    >     echo "begin block"
    >     defer: echo "defer 1 from block";
    >     defer: echo "defer 2 from block";
    >     echo "end block"
    >   }}
    
    begin header
    begin block
    end block
    end header
    
    defer 2 from header
    defer 2 from block
    defer 1 from block
    defer 1 from header
~~~

The above function runs the command it's given at the end of the "current" block -- i.e. the one that `defer:` is called from, or the one whose header it's called from.  As in Go, commands are stacked and run in reverse order.  They do not have access to the variables or arguments of the header or block, and unless `eval` is used any parameter expansion in the command is done at defer-time, not run-time.

In order to support extensions like this, `scale-dsl` wraps the execution of the block stack in such a way that:

* exit codes from deferred commands are ignored, even if `-e` or an `ERR` trap is active
* the value of the bash special variables  `$?` and `REPLY` are the same after the deferred commands as they were at the end of the block or block header

~~~sh
    $ mess() { echo "Imma change everything"; REPLY=blah; return 99;}
    $ ~ set -e
    >   defer: set +e
    >   defer: eval 'echo $? "${REPLY[@]}"'
    >   defer: mess
    >   dsl:
    >   {{
    >     REPLY=(what up)
    >     return 42
    >   }}
    > echo "$?" "${REPLY[@]}"
    Imma change everything
    99 blah
    42 what up
~~~

In the above example, the block is run with `-e` in effect, and even though it returns a non-zero exit status, the deferred commands still run.  The first deferred command makes changes to `REPLY` and `$?`, but doesn't exit either, and its changes are reverted after the final deferred command disables `-e` again (so the final result of the block doesn't cause an exit).