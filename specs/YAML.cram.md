## Clean YAML Generation

Here, we verify that our DSL for YAML generation behaves the same way as dirtsimple/clean-yaml for structures, and symfony/yaml for scalars (other than multi-line strings.

~~~sh
    $ source "$TESTDIR"/../demos/scale-yaml
    $ source scale-dsl
    $ show() { "$@"; echo -n "$REPLY"; }
~~~

### One-Liners

Top-level leaf values are inlined with a terminating LF:

~~~sh
    $ show yaml str xyz
    xyz

    $ show yaml null
    null

    $ show yaml map
    {  }

    $ show yaml list
    [  ]

~~~

That includes multi-line strings, since many YAML parsers (including Symfony's) don't support block literals at the top level:

~~~sh
    $ show yaml str $'the quick\nbrown fox\n'
    "the quick\nbrown fox\n"
~~~

And non-leaf, non-root data structures are inlined if they fit:

~~~sh
    $ ~ show yaml map; {{
    >   + xyz list; {{ - key $'the quick\nbrown fox\n'; - true; }}
    > }}
    xyz: [ "the quick\nbrown fox\n", true ]

    $ ~ show yaml --width 20 map; {{
    >   + xyz list; {{ - key $'the quick\nbrown fox\n'; - true; }}
    > }}
    xyz:
      - "the quick\nbrown fox\n"
      - true

    $ ~ show yaml --width 10 list; {{ + map; {{ - a str b; }}; }}
    - { a: b }

    $ ~ show yaml --width 8 list; {{ + map; {{ - a str b; }}; }}
    - a: b
~~~

But non-empty root structures are never inlined, no matter how easily they'd fit:

~~~sh
    $ ~ show yaml list; {{ - int 42; }}
    - 42

    $ ~ show yaml map; {{ - a str b; }}
    a: b
~~~

### Quoting

Strings aren't quoted unless they need to be, and single-quote strings are used if possible.

~~~sh
    $ show-strings() {
    >   printf ' '; for s; do yaml str "$s"; printf ' %s' "${REPLY%$'\n'}"; done; printf \\n
    > }

# Strings that don't contain structure delimiters, whitespace, control characters,
# quotes, or begin with markers don't need to be quoted in YAML:

    $ show-strings \
    > abc.def_ghi-xy/z  "jk<>l!%m@n" o99 ^q=r\|st~u
      abc.def_ghi-xy/z jk<>l!%m@n o99 ^q=r|st~u

# But everything else need at least single quotes:

    $ show-strings \
    > 'a,b' '22' '--' '<m' '>n' '=o' '|p' 'x:' 'v{' 'a b' 'q}' '.[' '_]' '^&' 'x?' 'e#' '"q"' "x''y"
      'a,b' '22' '--' '<m' '>n' '=o' '|p' 'x:' 'v{' 'a b' 'q}' '.[' '_]' '^&' 'x?' 'e#' '"q"' 'x''''y'

# Including reserved words:

    $ show-strings null yes no y n on off true false '~'
      'null' 'yes' 'no' 'y' 'n' 'on' 'off' 'true' 'false' '~'

# And control characters need double quotes:

    $ show yaml str $'xyz\r\nabc\r\n'
    "xyz\r\nabc\r\n"
~~~

### Text Chomping And Indents

~~~sh
    $ strlist() { ~ show yaml "${@:2}" list; {{ - str "$1"; }}; }

# A string with one \n that fits in the width isn't treated as a block

    $ strlist "This string ends with a '\n'"$'\n'
    - "This string ends with a '\\n'\n"

    $ strlist " This string ends with a '\n' and starts with a space."$'\n'
    - " This string ends with a '\\n' and starts with a space.\n"

    $ strlist "This string does NOT end with
    > a '\n', but it does contain one"
    - "This string does NOT end with\na '\\n', but it does contain one"

# But if it doesn't fit, or has at least two line feeds, it is:

    $ strlist "This string ends with a '\n'"$'\n' --width 30
    - |
      This string ends with a '\n'

    $ strlist " This string ends with a '\n' and starts with a space."$'\n' --width 30
    - |2
       This string ends with a '\n' and starts with a space.

    $ strlist "  This string also ends with a '\n' and starts with two
    > spaces, but has a different indent width."$'\n' --width 30 --indent 4
    - |4
          This string also ends with a '\n' and starts with two
        spaces, but has a different indent width.

    $ strlist "This string does NOT end with
    > a '\n', but it does contain one" --width 50
    - |-
      This string does NOT end with
      a '\n', but it does contain one

    $ strlist "This string ends with two '\n' characters"$'\n\n'
    - |+
      This string ends with two '\n' characters
      

    $ strlist " And so does this one."$'\n\n'
    - |2+
       And so does this one.
      
~~~

### Miscellaneous Checks

~~~sh
# Empty list inlined in an inlined map -- was duplicating the map key

    $ ~ show yaml list; {{ + map; {{ - x list; }}; }}
    - { x: [  ] }

# Align list items that are maps based on indentation

    $ ~ yaml_indent='   ' yaml_width=10 show yaml list; {{
    >   + map; {{
    >     + x list; {{ - str foo; + list; {{ - str bar; }}; }}
    >   }}
    >   + map; {{
    >     - w str foo
    >     - z str bar
    >   }}
    > }}
    -  x:
          - foo
          -
             - bar
    -  w: foo
       z: bar

# Multiline strings inside structures

    $ ~ show yaml list; {{ - str $'the quick\nbrown fox\n'; - int 42; }}
    - |
      the quick
      brown fox
    - 42

    $ ~ show yaml map; {{
    >   + xyz list; {{
    >     - str $'the quick\nbrown fox\n';
    >     + map; {{ - blue int 42; - 22 true; }};
    >   }}
    > }}
    xyz:
      - |
        the quick
        brown fox
      - { blue: 42, '22': true }

# Narrower width at a specific point

    $ ~ show yaml map; {{
    >   + xyz --width 12 list; {{
    >     - str $'the quick\nbrown fox\n';
    >     + map; {{ - blue int 42; - 22 true; }};
    >   }}
    > }}
    xyz:
      - |
        the quick
        brown fox
      - blue: 42
        '22': true

~~~

### Exact Width Inlining

This data structure should exactly fit in a width of 25, but not 24:

~~~sh
    $ width-test() {
    >   ~ show yaml --width "$1" map; {{
    >     + abc map;  {{ - b str c; - d str e; - f str g; }}
    >     + def list; {{ - str q; - str r; - str s; - str t; - str u; - str v; }}
    >     + ghi map; {{
    >       + j map;  {{ - x str z; - z str a; - a str b; }}
    >       + k list; {{ - str q; - str r; - str s; - str t; - str u; - str v; }}
    >     }}
    >   }}
    > }

    $ width-test 25
    abc: { b: c, d: e, f: g }
    def: [ q, r, s, t, u, v ]
    ghi:
      j: { x: z, z: a, a: b }
      k: [ q, r, s, t, u, v ]

    $ width-test 24
    abc:
      b: c
      d: e
      f: g
    def:
      - q
      - r
      - s
      - t
      - u
      - v
    ghi:
      j:
        x: z
        z: a
        a: b
      k:
        - q
        - r
        - s
        - t
        - u
        - v
~~~

