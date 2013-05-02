Short essays on selected Lua topics, including LuaTeX.

<style type="text/css">code{white-space: pre;}
body{font-family: Lucida, Times, serif;}
h1,h2,h3,h4{font-family: Lucida, Palatino, serif;
</style>


Brief introduction to LPEG
--------------------------

LPEG is a library for operating on patterns. It is not an alternative to the Lua string library, but it can be used to implement libraries rather like the string library instead of doing it directly in C.

Some words mean very specific things to LPEG.

string  
A Lua string, consisting of an arbitrary sequence of bytes. On many systems, one could say "characters" instead.

pattern  
A userdata type, containing enough information to characterize a certain property that a string might have. The properties that can be described are rather like the questions solvers of crossword puzzles tend to ask. For example: "seven letters, starts with 'p', has a double letter in it somewere".

subject  
A string that is presented for examination to an LPEG function. Its bytes are referred to as `input`.

match  
Not the same as `string.match`.

1.  If a string has the property determined by a pattern, the pattern is said to *match* the string. I.e. it is the pattern, not the string, that does the matching.
2.  The name of an LPEG function that applies a pattern to a subject in an attempt to find a matched substring.
3.  The string matched by a pattern, especially when it is only a substring of the subject.

The central function of LPEG.

success  
A pattern *succeeds* when it matches a substring of the subject at the point where it is applied.

failure  
A pattern *fails* when it does not match any substring at that point.

capture  
1.  One of the individually accessible Lua values returned by some patterns, e.g. a portion of a match.
2.  To return such a value.
3.  A capture pattern.

capture pattern  
A pattern that returns one or more captures when it succeeds.

consume  
A pattern *consumes* its match if that portion of the subject is not available to any follow-up pattern except a backspace pattern.

The most spectacular feature of LPEG is the way complicated patterns are built up from simpler ones: *Patterns can be used instead of numbers as the values that the variables in an arithmetic expression may take*. For example, "x\^2+3\*x-13" is a perfectly valid LPEG expression when `x` is a pattern.

Roberto Ierusalimschy put a lot of thought into allocating useful meanings to the arithmetic operations and constants of Lua. In particular, the priority of operations is such that one often needs no parentheses. Some words of warning, though:

-   If you use constants like `3` or `true` in a pattern expression, they cannot be combined with each other, only with existing patterns. To make sure, convert them to patterns first by applying `lpeg.P` (the `lpeg.` is used only here, it will be just `P` later).
-   The expressions look familiar but that must not mislead you into thinking that the usual commutative, associative and distributive laws of algebra hold. They don't.

Nil is not a valid pattern.  
This goes without saying, but I am nevertheless saying it.

Booleans denote success or failure.  
`true` stands for a pattern that always succeeds, `false` for a pattern that always fails.

Strings correspond to themselves.  
A string stands for a pattern that matches only that exact string.

Integers correspond to string length.  
`0` stands for a pattern that matches the empty string, positive `n` for a pattern that matches exactly `n` bytes, `-n` for the negation of `n` (see below).

Negation means reversal of fortune.  
`-p` succeeds when `p` fails. It consumes no input. The idiom `-1` matches only the end of the subject.

Length means don't consume.  
`#p` matches what `p` matches, but consumes no input.

Multiplication corresponds to concatenation.  
Suppose `p` and `q` respectively match `a` and `b`, then `p*q` matches `a..b`. Note that multiplication is not commutative.

Addition corresponds to shortcut `or`.  
`p+q` matches what `p` matches, except when `p` fails; then it matches what `q` matches. Note that `p+q` succeeds if and only if `q+p` succeeds, but if both `p` and `q` would succeed, the match is that of the first pattern. So addition is not quite commutative.

Subtraction corresponds to reverse shortcut `and not`.  
`p-q` fails if `q` succeeds, otherwise matches what `p` matches. Note that `0-p` does the same as `-p`, but `p-p` does not do the same as `0`, it does the same as `false`.

Division means use the captures.  
`p/s` matches what `p` does, and defines a capture for the whole expression by processing the captures of `p` as specified by `s`. Call the captures `c_1`, `c_2` etc. If `p` defines no capture, the whole substring matched by `p` counts as `c_1`. There are four variations, depending on the type of `s`.

-   number: `p/n` means the capture is `c_n`.
-   table: `p/tbl` means the capture is `tbl[c_1]`.
-   string: `p/str` means the capture is the result of replacing in `str` all instances of `"%1"` by `c_1`, `"%2"` by `c_2`, etc. This feature can only handle nine captures.
-   function: `p/fct` means the capture (or perhaps several captures) is `fct(c_1,c_2,...)`.

This is the easiest way to make a pattern that returns captures. There is much more below under [Captures](#captures).

Exponentiation corresponds to repetition.  
Not quite exponentiation in the usual sense: `p*p` means *exactly* two repetitions of `p`, which is not the same as `p^2`.

-   `p^n` matches `n` or more repetitions of `p`.
-   `p^0` matches any number of repetitions of `p`, including the empty string.
-   `p^-n` matches not more than `n` repetitions of `p`.

A common idiom: `#p*q` matches what `q` matches, provided that `p` succeeds.

For example, suppose `x=P"abc"`. Then `x^2+3*x-13` means "two or more copies of `abc`, or any three bytes followed by `abc`, but not 13 or more bytes long".

Some LPEG functions
-------------------

The introductory `lpeg.` has been omitted here.

### Pattern constructors

`P` for Pattern  
Apart from nil, boolean, string and number, discussed above, functions and tables can also be converted to patterns, but these are too advanced to discuss here. Existing patterns are unchanged.

`R` for Range  
`R(r)`, where `r` is a two-byte string, matches any byte whose internal numerical code is in the range `r:byte(1,2)`. You could use characters in `r`, e.g. R"az" on most systems matches the range of lowercase letters, but see `locale` for a more portable alternative.

`S` for Set  
S(s), where `s` is a string, matches any of the bytes in `s`.

`locale`  
`locale()` returns a table of patterns that match character classes. Recommended method is to examine the keys of the returned table, e.g. `locale().lower` matches all lower-case letters.

### Pattern methods

When called with `p` as first argument, these functions will replace `p` by `P(p)` before proceeding. When `p` is already a pattern, they can also be called the object-oriented way shown below.

`match(p,subject[,init])`, `p:match(subject[,init], ...)`  
Tries to match `p` to `subject:sub(init)`. `init` defaults to 1. Returns the captures of the match, or if none specified, the index of the first byte in the subject after the match. The optional arguments can be accessed by the functions described under [Captures](#captures).

`B(p)`, `p:B()` for Backspace  
Matches what `p` does, but matches just before the current position in the subject instead of at it and consumes no input. `p` is restricted to patterns of fixed length that make no captures.

`C(p)`, `p:C()` for Capture  
Matches what `p` does, and returns a capture of the match, plus all captures that were made.

Advanced topics
---------------

### Captures

The patterns made by `P` acting on a non-pattern do not capture anything. They can be made to do so with the four options provided by the division operator. In principle one can do almost anything that way, since one of the options is to process everything by a function of one's own choosing.

Captures made by a subexpression do not automatically become captures of the whole expression. The division operator and the capture methods must be explicitly invoked in order to pass captures from a pattern to the expression containing it and eventually to the output of `match`.

In practice some tasks are so common that it is useful to have additional capture functions predefined. We are reaching the limits of this primer here, and there is really no substitute for reading the official LPEG documentation, but at the risk of saying too much but not enough, here are some of them.

`Carg(n)`  
Captures the `n`-th extra argument to `match`.

`Cp()`  
Captures the position `n` in the subject that has been reached. I.e. the input up to position `n-1` has been consumed when `Cp()` is matched.

`Cc(...)`  
Captures all its arguments.

`Ct(p)`, `p:Ct()`  
Captures a table containing all the captures of `p`.

-   There are several specialized constructors and methods dealing with captures.

-   `P` acting on tables and functions.

-   The LPEG distribution also contains `re.lua`, an application demonstrating the feasibility of writing a regular expression handler using LPEG.

The present author is not yet qualified to write about these.

* * * * *

\<- *About this document:* Dirk Laurie wrote it in order to teach himself LPEG. All errors in it can be blamed on his inexperience. The original is `lpeg-brief.txt`, which is written in Pandoc Markdown. The HTML version was made using Pandoc <http://www.johnmacfarlane.com/pandoc>.
