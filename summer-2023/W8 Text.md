## Problem Of Text

* Want to represent text (written material, in broad sense)
  in a computer.

* *Much* trickier than it appears.

  * Multiple languages.
  * Non-alphabetic languages.
  * Odd "symbols".
  * Fonts and "styles".
  * General typography.
  
## Basic Approach To Text

Split into representation hierarchy.

* Document collections, "documents": high-level
abstractions.
* "Texts": incorporate formatting — fonts, styles,
typography.
* "Strings": Little language representing a sequence
of "characters".
* "Characters": Atomic unit of text. May be its own little
language.

## Characters: ASCII

* Obvious character representation: as bytes. ASCII is the
  canonical byte representation.
  
* Some typography crept into ASCII:

  * Space
  * Tab
  * "Carriage Return" / "Line feed" ("newline")
  * Other "control characters"

* ASCII is 7-bit (!): this is both an opportunity and a danger.

* ASCII is impossibly limiting for the modern world.

## Characters: Unicode

* Technically, "code points". Composed into "grapheme
  clusters". Ugh.

* Result of 1990s standardization effort.

  * Originally two-byte encoding — 65536 code points.
  * Then two bytes was found to be not enough, so 21-bit
    encoding (17 × 2**16) usually stored in four bytes.
  * Lower 128 code points are exactly ASCII.

* Goal is to represent every symbol that is used in
  real-world writing. This includes… a lot.

## Characters: ISO Encodings

* Nope. Just… nope.

## Characters: Windows Code Pages

* Nope. Just… nope.

## Characters: Old Non-Western Encodings

* Nope. Just… nope.

## Strings

* In ASCII world, an ASCII string is just a sequence of
  bytes representing ASCII characters.

* Still have to decide where the string ends. Obvious
  choices:
  
  * Give a length field up front to indicate how many bytes
    are coming. Limits string length depending on length
    encoding. Popular choices: unsigned 8-bit, 16-bit,
    32-bit, 64-bit length field.

  * Have some kind of string terminator "character". Maybe
    make it escapable. Maybe do what C/C++ do and just make
    NUL (ASCII 0) unrepresentable in strings.

## Unicode: Strings

* In Unicode, the obvious choices would lead to strings
  being represented by sequences of 32-bit Unicode code
  points. This is highly inefficient for mostly-Western text
  (which tends to use the lowest 256 codepoints). It also
  has an annoying "endianness" problem.

* Worse, Unicode strings predate 21-bit Unicode: they were
  designed during 16-bit Unicode times. So a common
  representation was as 16-bit (LE or BE) code points.
  
  * This was "solved" for 21-bit by making the codepoints
    above 65536 be represented by an escapish pair of
    codepoints called a "surrogate pair": the codepoints
    0xd800-0xdbff are the "first half" and allow encoding
    10 bits of the target. 0xdc00-0xdfff are the second half
    and can encode another 10 bits. Add 0x10000 to the
    20-bit result to get the "real" codepoint.
    
  * Remember, endianness matters.

## Unicode: Combining Characters

* Not economical to represent, for example, every letter
  with a tilde above it (x̃) as a separate codepoint. So
  "combining" codepoint can follow a character
  in a string.
  
  Extreme case is Z̴͈̺̖̈́̋̉ͅä̶͔̣̟̦͖͈͈̮̺̊̋͑̾̇͗̒l̷̺̙̂̇͑̂̿̏͛͝g̷̢̧̫̠̰̜̠̦̬̭̜̳̤͉͈̑͛̓̄́̅̊͐̅́̕̚ͅo̷̧͕̤̮̺̤͈͚̟͚͋̏̈̉̓̉̽͒̿̄͆̓͊͜͜͝͠

* Some characters (ñ) have both a single codepoint and
  combining character representation.

## Unicode Normal Forms: DNF, CNF

* ñ and ñ may not be the same codepoints depending on
  representation: one might be the code for "tilde n" and
  the other the two codes for "n, combining tilde above".
  
  Thus, comparing two strings byte-for-byte might produce 
  false negative.
  
* Unicode defines a "normalization" algorithm for
  "decomposing" a string into combining codes as much as
  possible, producing a "Decomposed Normal Form" (DNF) that
  can be compared byte-for-byte with other DNF strings.
  
* Unicode defines a "normalization" algorithm for
  "composing" a string to remove combining codes as much as
  possible, producing a "Composed Normal Form" (CNF) that
  can be compared byte-for-byte with other CNF strings.
  
* Most things don't care. The remainder usually use CNF
  because it is potentially slightly smaller.

## Unicode Normal Forms: DKNF, CKNF

* There are code points that are different for various
  technical reasons, but "mean the same thing" (f, ﬀ, 𝖋).
  
  Unicode "Compatibility" (K, because C for composition) normalization
  algorithm reduces these to "least" form so that they can 
  be compared (f, ff, f).

* This is independent of CNF/DNF, so CKNF/DKNF.

## Unicode Strings: UCS-2, UCS-4, UTF-16, UTF-32

* Unicode string efficiency for Western alphabets is so far
  pretty mediocre:
  
  * UCS-2 represents only two-byte code points: garbage now
  * UTF-16 represents Western code points with two bytes,
    other code points with four-byte surrogate pairs.
  * UCS-4 / UTF-32 (distinction is not much relevant)
    represents every code point by four bytes. Ugh.

    (Why no UTF-24 / UCS-3?)

* Storing / scanning all those zeros is silly expensive.
  Maybe some "compressed" encoding would be better?

## UTF-8

* The "one true" Unicode string format.

* ASCII code points represent themselves. Upper bit is used
  as an escape to represent bigger code points.
  
* Escapes with more bytes used to represent code points with
  more significant bits. Maximum 5 bytes for a high code
  point.

* Rust string representation: `usize` length, then UTF-8
  bytes.  No specific normal form: normalize yourself (using
  a crate implementing Unicode algorithm) as desired.

## String Escapes

* Idea: Use some sequence of characters to represent a
  single character.
  
* Have already seen this idea in UTF-16 / UTF-8.

* A "good" "escape convention" allows all possible
  characters to still be represented. (So the C NUL length
  thing that has no escape is bad. Should have used
  NUL/count or something that treats two NULs in a row as
  end-of-string.)

## Postgres and Unicode

* Postgres allows different string representations:
  representation is chosen at database creation time.
  
* Default today is UTF-8. Older databases might have almost
  anything.

* How to store Unicode string in ASCII-only string column?

* Postgres allows Unicode escapes like `\u008a` to mean
  "codepoint 0x8a". Can escape the backslash: `\\`. Can
  escape string quotes: `'don\'t'`.
  
* This is "safer" when working with an unknown database.

* Postgres database driver and client normally cooperate to
  handle this.

## JSON and Unicode

* JSON allows UTF-8. It also allows backslash escapes,
  including Unicode code point escapes.
  
* Serde handles the translation back and forth just fine.

## We're Good Now, Right?

* Heck no. We have *strings*, mostly.

* Unicode has higher-level stuff to support text direction,
  internationalization, etc.

* A "text shaping engine" (usually HarfBuzz) handles this
  stuff plus higher text concepts like "font", "paragraph",
  "bold", etc.

* A "layout engine" places shaped text in an area.

* Browsers handle this in web apps, with CSS controls for
  it. Mostly.

## Text Is A Problem

* So much to get wrong. 

* So much to learn to get stuff right.

* "Good enough" is slippery.
