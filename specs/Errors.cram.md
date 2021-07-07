## Error Handling

Errors in blocks should not be suppressed if `set -e` is in effect

~~~sh
    $ . scale-dsl
    $ (set -e; ~ ::block; {{ (exit 99); echo "should not reach here"; }}; )
    [99]
~~~

