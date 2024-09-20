# Goals

- Document the exact character class syntax of PCRE2
- Same for Perl, .NET, and JavaScript
- Study UTR#18

- Propose updated UTR#18-compatible classes

# Proposed new syntax

:( Basically, Perl's "extended" syntax is really quite different from every other regex dialect. It'll need its own parser, and it's not going to align with any other languages in the "top 10". Hum.

So what I'm proposing is that PCRE2 adopt a flag "extended classes" which makes ordinary `[...]` classes behave like the Python/.NET/JavaScript/UTR#18/etc classes. This will (as far as I can tell) become the new/future default syntax for character classes, in all engines _other_ than Perl.

Additionally, PCRE2 can of course also implement the different direction that Perl went down, with a flexible parser which also takes as input their `(?[...])` idea.

## Historical summary

* Java led the way. Perhaps the first regex engine to introduce this sort of syntax extension? Went with `[abc&&[cde]]`, using `[...]` for nested character classes, and `&&` for set intersection.
* .NET was also an early adopter. Went with `[abc-[cde]]` using single hyphen `-` for set subtraction.
* Perl went off in a completely different direction with its `(?[...])` thing. No other engine has followed its lead, for once. They use `(?[(abc) - (cde)])` as their syntax, with parentheses as the bracketing mechanism!
* Unicode's UTR#18 spec _initially_ (years ago) recommended the old Java/.NET behaviour, using single-character operators `&` and `-`.
* Then the world moved forwards - and UTR#18 now recommends the "modern" behaviour of `&&` and `--` operators, with `[...]` for nested classes.
* ECMAScript and Python (#1 and #2 most-used programming languages) have basically settled on the newest UTR#18 style.

## Proposed syntax for `[...]` in PCRE2, with `PCRE2_EXTENDED` flag

```ebnf
(* Note that this isn't 100% accurate. PCRE2 allows `\Q\E` and `\E` sequences before the `^`,
(* and with `(?xx)` also `SPACE | HT`. *)
CharacterClass := '[' '^'? ClassContents ']' ;

(* Non-empty contents, unless the PCRE2_ALLOW_EMPTY_CLASS flag is set. *)
ClassContents := ClassImplicitUnion
               | ClassIntersection
               | ClassSubtraction ;

(* A non-empty "implicit union" formed by concatenation of characters, escapes, ranges,
(* and also nested classes. So for example, `ab-d\d[^c]` is a legal ClassImplicitUnion. *)
ClassImplicitUnion := (ClassChar | ClassEscape | ClassRange | ClassPosix | CharacterClass)+ ;

(* A set-wise intersection of two character classes. Associativity irrelevant.
(* The left- and right-hand sides of the operator must be non-empty. *)
ClassIntersection := ClassImplicitUnion ('&&' ClassImplicitUnion)+ ;

(* Set-wise subtraction of a set of characters. Left-associative. *)
ClassSubtraction := ClassImplicitUnion ('--' ClassImplicitUnion)+ ;

(* ClassChar and ClassRange have a lookahead rule so that `--` and `&&` are never
(* parsed as characters. *)
```

Some remarks:
* We do allow implicit unions in the presence of operators. This is looser than ECMAScript. For example, we allow `[b-d&&ac-k]` where the operators can have ranges on their left and right, and unions. ECMAScript requires this to be written as `[[b-d]&&[ac-k]]`. They allow single characters next to operators such as `[a&&[c-d]]`; but not unbracketed ranges `[a&&c-d]`, or a union of characters and ranges `[ac&&d]`.
* We do not specify any precedence between `--` and `&&`. A `ClassContents` cannot contain both - `[a&&b--d]` is illegal, as it is in ECMAScript. There is too much disagreement across the ecosystem on whether the operators all have the same precedence, or whether some bind more tightly. Like ECMAScript, we should avoid committing on this.

  Realistically, PCRE2 is not a "trend-setter" to the extent that JavaScript or Python or Rust or Ruby are going to follow us (although I'd hope they at least take note of PCRE2's syntax). So making a guess on operator precedence, which in ten years turns out to be inconsistent with where the rest of the software ecosystem ends up, would be rather unfortunate.

  I think it's OK to require brackets in `[[a&&b]--d]` or `[a&&[b--d]]` if you want to mix operators.

  There's nothing stopping PCRE2 from widening its syntax in future.
* What's different from current `[...]` syntax in PCRE2? What stops this becoming the new default in PCRE2, with a `PCRE2_UNEXTENDED_CLASS` flag for the old behaviour?
  * Requires a `\[` to escape opening square bracket inside a class. Previously this was fine without escaping. This will definitely trip up users in practice.
  * Doesn't allow you to use `&&` and `--` (but why would you anyway). Previously you could write a character range such as `--A` for "characters from HYPHEN to A". I think this is a purely theoretical backwards-incompatibility.
  * So - fairly minimal, to be honest.

This syntax proposal is compatible with all the UTR#18-inspired regex engines surveyed below, in the following sense:
* Any regexes accepted by both this PCRE2 proposal and other libraries behave the same.
* The accepted regexes are more permissive than ECMAScript; more restrictive than others like Rust.
* The engines like .NET that predate the use of double-character operators in UTR#18 are excluded from comparison. They'll catch up one day... maybe.

# Background details which inform the proposal

## Detailed parsing of existing PCRE2 syntax

- Naturally, starts with an opening `[`, closes with `]`
- Does a one-character lookahead for `^`
- Does a one-character lookahead for `]` which may appear unescaped as the first character of the class, unless the `PCRE2_ALLOW_EMPTY_CLASS` flag is set.

  In most dialects, `[]abc]` is the class containing `]abc`, but in JavaScript (the weird one...) they don't special-case the first character of the class, so it's instead parsed as the empty class followed by the literals `abc]`.
- There are also some extended behaviours around the special-casing of the first character as `^`. In most dialects, it really _must_ be literally the first character in the class, but, PCRE2 has some do-nothing syntaxes such as `\E` (harmless redundant end-literal), `\Q\E` (literal sequence of zero characters), and surrounding whitespace in `(?xx)` mode. So, the hunt for `^` is mixed with skipping of these.
- The rules for matching a POSIX class are complicated...

  - Check for `[:` or `[.` or `[=`
  - Scan for the matching close, processing simple backslash escapes within the POSIX name
  - An unescaped `]` may not appear in the POSIX name
  - An unescaped POSIX start may not appear in the POSIX name
    - XXX should it scan for _any_ of the POSIX start sequences? Currently it scans for whether `[:` appears inside a `[:...:]` POSIX class, with matching terminators only.
  - Syntax:
    
    ```ebnf
    PosixClass := '[.' PosixContents*? '.]'
                | '[:' PosixContents*? ':]'
                | '[=' PosixContents*? '=]' ;
    PosixContents := '[' if lookahead NOT [.:=]
                   | '\\' | '\]'
                   | any character not ']' ;
    ```

- Note that the special sequences `[[:<:]]` and `[[:>:]]` are excluded from character class processing.
- Also, forbid a `CharacterClass` from being a `PosixClass` (so `[[:alnum:]]` is OK but `[:alnum:]` is illegal syntax rather than a character class including COLON).

- Finally, the actual parsing of the `ClassAtom` and `ClassRange := ClassAtom '-' ClassAtom` constituents is highly complicated as a BNF grammar, because the grammar's precedence must encode the priority of hyphen as a literal and as a range separator. You can write `[--A]` for "the characters from HYPHEN to A".

  It is clearer to describe the class contents imperatively:
  - A class is a sequence of

    ```ebnf
    ClassAtomNoRange :=
          '\Q' .*? '\E' (* actually not 100% correct... each character inside \Q...\E is an atom *)
        | PosixClass    (* possibly negated... ALSO we recognize & fault [.ch.] and [=ch=] *)
        | '['           (* not a metacharacter at all *)
        | ClassEscape
        | not ']' or '-' ;
    ClassAtom := ClassAtomNoRange | '-' ;
    ClassRange := ClassAtomNoRange '-' ClassAtomNoRange ;
    ```
  - POSIX ranges are treated the same as `\d` and `\w` etc. They are parsed as range ends when they appear as `[\d-z]` but the range is then faulted because both ends of the character range must be a single character. The hyphen is NOT treated as a literal surrounding `\d` or `[:alnum]`.
  - PCRE2 implements hyphen rules with some very, very fiddly lookahead. Urgh. After POSIX or `\d` classes we peek for no following hyphen except for the case of `-]` closing the class.
  - Peek for HYPHEN on each loop. If we have just consumed a character, then initiate a range; otherwise it's a literal.
  - Check for BACKSLASH on each loop, and handle.
    - Characters which are not 0-9 A-Z a-z simply escape to themselves.
    - Common character escapes: \a \e \f \n \r \t
    - Also `\b` for BACKSPACE (historical)
    - \x{...} and \U
    - octals, \c?
    - XXX/FIXME Why do we allow a quantifier following \N in class mode? Surely that check should be skipped in check_escape() when inside a character class?
    - \B \R \X excluded early on; why? Would it matter just to fall through, same as all the other escapes unsupported in a class?
    - \h \v \d \s \w and \p are like the POSIX classes: cannot be a range end.
  - Check for `]` closing the class.
  - Everything else is a literal.
    - If in a range, finish it.
    - Check range ends are sorted.
    - Special tracking of whether the ends of the range were produced via an escape or not (for EBCDIC).
    - After a range, a hyphen cannot be a range separator.
    - After a non-range, a hyphen is a range separator (...except at the end of the character class).
    - How to \Q...\E interact with ranges? Fascinatingly, the answer is... not at all. The \Q and \E themselves simply cause the characters contained within to be interpreted as literals, so `[\QA\E-Z]` is completely fine for specifying a range, as is `[A\Q\E-Z]` (however `[A\Q-\EZ]` is not a range, it's a literal hyphen). Also interestingly, \Q...\E doesn't set the "was escaped" flag for the EBCDIC interpretation of ranges.
  - On finishing the class, if we have an open range, then go back to fix it up to be a trailing hyphen.


This logic is way over-complicated by the fact that it's not using any function calls; therefore the main for loop can only process one atom (character) at a time, requiring multiple duplicated copies of logic to peek ahead and validate that ranges are well-formed.

This is how the logic would be, if main loop used a function call instead of one loop with gotos. Note that we don't need any logic for `class_range_state`. I believe this is equivalent to PCRE2's current logic:

```c
const char* ptr;
const char* ptrEnd;
assert(ptr < ptrEnd && ptr[0] == '[']);
++ptr;

// Special start-of-class meaning for ^
if (ptr < ptrEnd && ptr[0] == '^') { negated = true; ++ptr; }

// Special start-of-class interpration of ]
if (ptr < ptrEnd && ptr[0] == ']') { pushAtom(']'); ++ptr; }

// Main loop
while (ptr < ptrEnd && ptr[0] != ']')
{
    // Parse an atom. The next character must be an atom, since we're not at the end
    // of the class.
    int atom1;
    int err;
    if ((err = parseAtom(&ptr, ptrEnd, &atom1)) != 0)
        return err;
    
    // Now we do the peek for a range.
    // Only need to do the lookahead in one single place
    if (ptr == ptrEnd || ptr[0] == ']')
    {
        pushAtom(atom1);
        break;
    }
    if (ptr[0] == '-' && (ptr + 1 == ptrEnd || ptr[1] == ']'))
    {
        pushAtom(atom1);
        pushAtom(charHyphen);
        ++ptr;
        break;
    }

    if (ptr[0] == '-')
    {
        // We know this hyphen is a range separator now.
        ++ptr;

        // This is WAY more readable than PCRE2's current code, because here we have
        // a second call to parseAtom(). PCRE2 current inlines parseAtom into the for
        // loop, so it can't be called twice, and needs additional state outside the
        // for loop to track the class ranges.
        int atom2;
        if ((err = parseAtom(&ptr, ptrEnd, &atom2)) != 0)
            return err;
        
        // Can't have \d on either end of a range. Must be single characters, with
        // equal or ascending codepoint value.
        // BAD: [b-a], [\d-a]
        // OK:  [a-a], [a-b]
        if (!isOkRangeEndpoint(atom1) || !isOkRangeEndpoint(atom2) || atom2 < atom1)
            return ErrBadRange;
        
        pushRange(atom1, atom2);
    }
    else
    {
        pushAtom(atom1);
    }
}

if (ptr == ptrEnd)
    return ErrUnclosed;
// The main loop continues until it reaches ']' or end-of-string
assert(ptr[0] == ']');
return OK;
```

## Survey of other regex dialects

### Perl

I have spent some time trying to decode the C source code for Perl. It's ... not as tidy as PCRE2! OK if you want to hunt for some tiny specific detail in isolation, but hard to grok how the larger pieces fit together.

So I fell back on the documentation: https://perldoc.perl.org/perlrecharclass

Unsurprisingly, what they _describe_ for simple `[...]` classes matches PCRE2's behaviour. Of course, without a source code audit, I can't guarantee that what they _implement_ matches PCRE2's behaviour.

They also describe their "extended" syntax available under `(?[...])`.

- Completely unlike the UTR#18 extended character classes, Perl's one enables the `(?xx)` mode always.
- Operators `&` (intersection), `+` (union), `|` (alias for `+`!!), `-` (difference), `^` (XOR, symmetric difference), and `!` (unary complement).
- `!` has high precedence as expected; then `&`; then the others at equal precedence. All left-associative.
- It looks like `()` are semantic parens, ie used to control order of operators; and that `[]` are like "quotes" that revert to ordinary character class syntax inside, so that `[()]` means "either ( or )" when inside an extended class.

### UTR#18

Link: https://www.unicode.org/reports/tr18/

For PCRE2, Perl is obviously the #1 reference point to which to match behaviour.

But UTR#18 is probably the next-best thing, describing what the Unicode Consortium thinks is correct Unicode handling in regexes.

It's a pretty vague standard actually, with a lot of wiggle room. On one hand, that's nice: let the regex engines do "regex stuff", while Unicode just specifies how characters function (inside regexes).

But on the other hand, that's super annoying, because they propose new regex syntax without properly specifying it, leading to different programming languages and engines adopting the UTR#18 proposals with different choices! Maddening.

Their syntax:
* Specify `[...[...]...]` nested character classes. Changes the meaning of `[` inside classes to be a metacharacter requiring escaping.
* Adds operators `||` `&&` `--` `~~` and hand-waves about implementations having different ideas on what syntax is allowed on either side of them, and whether you can do `[a&&b--c]`.
* Requires "some syntax for union" such as `[AB]` or `[A||B]`, but doesn't require the `||` operator, just suggests you might want to provide it.
* Requires `&&` and `--`, and describes `~~` symmetric difference, but doesn't say you need to provide it.

### Python - built-in `re` module

Has no extended classes yet; but does emit a `FutureWarning` if you use unescaped `[` or `--`, `&&` etc inside a character class.

As far as UTR#18 goes, they haven't implemented it yet, but made a clear statement that they are going down that route in future, by deprecating use of unescaped `[` inside classes, so they can make it a metacharacter later on.

Parser code: https://github.com/python/cpython/blob/8ac085579acbd74d251985b77e13efe30d3d59cc/Lib/re/_parser.py#L549

Mercifully short & sweet source code! Far easier to follow than PCRE2 (benefits of a high-level language, and much much simpler syntax - no `\Q...\E` or POSIX classes).

Pseudocode for their parsing of when HYPHEN is a literal, and when it's a character-range separator:

```python
items = []

def read_atom():
    if peek_char() == '\\':
        return consume_escaped()
    else
        return consume_literal()

while peek_char() != None:
    atom1 = read_atom()
    
    if peek_char() == '-':
        if peek_char() == None:
            raise ErrUnterminated
        if peek_char() == ']'
            items.push(atom1)
            items.push('-')
            break
        
        atom2 = read_atom()
        # ... validate range
        items.push(Range(atom1, atom2))
    else
        items.push(atom1)

return items
```

### Python - `regex` module

Link: https://pypi.org/project/regex/

This is a third-party PyPi module, which is like the "experimental" version of the core standard library `re` module. Features in here are not guaranteed to graduate into the standard library engine; however it would be highly surprising and rare if features were adopted with incompatible syntax or behaviour.

To put it another way, where `regex` goes, `re` is forced to follow unless there's a very good reason.

They added all of the operators from UTR#18: `||`, `~~`, `&&`, `--`. They allow complex unbracketed expressions such as `[ac-e&&f--z]`, with precedence of operations increasing in that order, and juxtaposition (implicit union) having tightest/highest precedence.

So, it's pretty permissive, and somewhat opinionated, by committing to a precedence order.

### JavaScript

Link: https://tc39.es/ecma262/multipage/text-processing.html#sec-regexp-regular-expression-objects

Their discussion of the rationale for their design (highly instructive):
https://github.com/tc39/proposal-regexp-v-flag

* Have added `[` as a metacharacter, but ALSO banned a load of other characters from appearing unescaped in character classes. All of the following are forbidden without escaping: `( ) [ ] { } / - \ |`. I guess they felt badly bitten by the whole backward-compatibility fiasco of adding `[` as a new metacharacter inside classes, and didn't want that ever to happen again, so they went and banned use of all regex metacharacters inside classes!
* Also added the `[\q{...}]` thing, that lets you put multi-character strings inside a character class. Yes! That's a Unicode thing from UTR#18.
* Reserved all ASCII double-punctuators such as `##` or `$$` or `,,` inside character classes, for use in future operators.
* Implemented just `--` and `&&` operators.
* Disallow any precedence between operators. Must be fully-bracketed, but can have multiple of same operator `[a--b--c]`.
* Cannot have an implicit union next to an operator, or a range. BAD: `[ab--c]` and `[a-c&&d]`. But `[[X]--[Y]]` allowed, and `[x&&y]`.

There's nothing bad here. It's just much stricter than PCRE2 needs to be, I believe.

Addendum - see their long discussion on the Unicode behaviour of `/[^\P{...}]/i`. Worth checking what different engines do, and make sure we are in alignment. https://github.com/tc39/proposal-regexp-v-flag/issues/30

### .NET

Link: https://learn.microsoft.com/en-us/dotnet/standard/base-types/character-classes-in-regular-expressions#character-class-subtraction-base_group---excluded_group

It's legacy syntax only. They have `[x-[y]]` for character class subtraction. It's not a fully-fledged implementation of nested character classes. The only operator is the single-character `-` HYPHEN and it requires brackets on the RHS and not on the LHS.

I'm sure they'll add something more compliant with UTR#18 whenever they can get to it, and work out a backwards-compatibility strategy.

The first-mover disadvantage!

### Java

Link: https://docs.oracle.com/en/java/javase/23/docs/api/java.base/java/util/regex/Pattern.html

Similar to .NET. It's legacy, as one of the first-movers to offer character set syntax.

They have some basic support for nested character classes, `[abc[d-f]]`.

They have one operator, `&&` for intersection, and require brackets on the RHS like Java.

### ICU (libicu)

Link: https://unicode-org.github.io/icu/userguide/strings/regexp.html

ICU always does what Unicode says. Their "unique selling point" is "we comply to more Unicode specs than whatever your standard library does".

So, it's no surprise that they have super-duper compliance with all the crazy stuff about character normalisation inside regexes!

And of course, they support UTR#18, because clearly that's more important than compatibility with any other regex dialect.

* Allow everything from UTR#18.
* Appear to allow mixing operators `[a-c&&b--a]`.

### Ruby

Link: https://docs.ruby-lang.org/en/3.3/Regexp.html

* Allows nested classes `[abc[0-9]]`
* Has one operator `&&`

Either they copied Java, or vice-versa?

### Rust

Link: https://docs.rs/regex/latest/regex/

It's not at all similar to PCRE2. They have a pure DFA engine, with guaranteed `O(m*n)` matching time (no backtracking at all). It supports capture groups, but not backreferences to them.

It's most similar to Google's RE2, which is another "never exponential" regex engine.

However, their syntax (unlike RE2) is very standard stuff.

* They say they support UTR#18 level 1. All the cool kids are documenting their Unicode compliance this way.
* They have added precedence to the operators... but differently to Python's `regex` module! Ouch.
  * Juxtaposition is tightest, `[ab]` for union
  * Then `&&` or `--` or `~~` are all at equal precedence. This is what UTR#18 suggests.