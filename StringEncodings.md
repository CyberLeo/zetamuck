# Introduction #

As far as MUCK has ever been concerned, a string is a string. Historically, the codebase has filtered out extended character sets as garbage using the `isascii()` function. The only potential source of garbage being rendered to the client was from the MUF debugger. That said, this was still a fairly rare event as usually it was a result of working with primitives interfacing with file or socket I/O.

Several years ago, that changed. In order to permit clients to send and receive bytes from the Latin-1 (ISO-8859-1) character set, ProtoMUCK decided to advertise itself as "8-bit clean" and disable the `isascii()` safeguards. While it was conceivably possible that a client might choose to have a different encoding in place (say, ISO-8859-2, Central European), in practice this was never the case with clients connecting to MUCK and no conflicting interpretations of those characters emerged.

# The Multi-byte Problem #

Times are changing again. Outside of the MU community, UTF-8 has become the most popular character set due to its ability to express UNICODE characters while remaining backward compatible with 7-bit ASCII. UTF-8 represents individual UNICODE characters as multi-byte sequence.

Example:
  * The ☺ character is known by its 16-bit representation 263A, or more commonly "U+263A".
  * The UTF-8 representation of this character is three bytes. Their hexademical representation would be e2 98 ba; one leading byte and two continuation bytes.

Up until now, MUCK has only had to deal with single byte encodings. We can derive three basic string manipulation problems from this:
  * The `STRCAT` MUF prim has functioned under the assumption that any freeform combination of bytes is legal...which may be true for binary and ISO-8859-1, but not UTF-8. It's no longer feasible for `STRCAT` to simply smash bytes together blindly.
  * `STRLEN` has always returned the length of a string. The ☺ character is _one_ UTF-8 character, but `STRLEN` would consider that to have a length of _three_ because it required three bytes to represent it. We now have to differentiate between length in bytes vs. length in characters.
  * `STRCUT` likewise assumes that it's no big deal to arbitrarily split a string in half. `"foobar" 1 strcut` yields the strings `"f"` and `"oobar"`, which is fine. Performing `☺ 1 strcut` would yield two garbage UTF-8 sequences. We now have to differentiate whether the index is referring to a byte position or a character position.



# String Compatibility #

Based on the lengthy intro above, we know we have several types of strings we're dealing with:

  * 7-bit ASCII
  * ISO-8859-1 (Latin-1)
  * UTF-8
  * Raw I/O (binary)

ASCII, ISO-8859-1, and Binary can be smashed together all day long because they're single byte encodings. The meaning of those individual bytes will differ (and may not be printable), but the end result is never something that would be considered illegal.

UTF-8 is the encoding that doesn't play well with others. Arbitrary combinations of ISO-8859-1 and binary characters are not guaranteed to be legal for UTF-8. Latin-1 characters can be converted to their UTF-8 counterparts 100% of the time. UTF-8 contains countless characters that _cannot_ be represented by Latin-1.

ASCII, to contrast, plays well with everyone. You can append ASCII characters to any of the encodings mentioned above and always have a valid string.

# Solution 1: Two types of strings #

  * All strings generated via socket prims or file prims are assumed to be RAW strings.
  * All strings obtained via MU clients are considered to be normal strings. (UTF-8 representations, converted from Latin-1 if necessary)
  * String operations on RAW strings operate on byte indexes.
  * String operations on normal strings operate on character indexes.
  * Combining a RAW string with a normal string is a run-time error. They must be converted with new primitives first.

Given the simplicity if this model, it's my current preference for implementation. It would be possible to implement the concept of raw strings without any UTF-8 specific coding, making it possible to test the implementation prior to overhauling the rest of the game to match.

# Solution 2: Four types of strings #

This is probably overkill.


Any string type appended to the same string type remains the same.

```
  ASCII +   ASCII =   ASCII
LATIN-1 + LATIN-1 = LATIN-1
    RAW +     RAW =     RAW
  UTF-8 +   UTF-8 =   UTF-8
```

Any string type combined with ASCII yields the non-ASCII string type. The ordering does not matter.

```
LATIN-1 +   ASCII = LATIN-1
  ASCII + LATIN-1 = LATIN-1
    RAW +   ASCII =     RAW
  ASCII +     RAW =     RAW
  UTF-8 +   ASCII =   UTF-8
  ASCII +   UTF-8 =   UTF-8
```

Combining a Latin-1 string with a UTF-8 string yields a UTF-8 string. This is because UTF-8 can represent Latin-1 characters, but the reverse is not consistently true.

Another possible implementation would be to convert the output to Latin-1 if the UTF-8 string consists entirely of characters that can be safely mapped to it, but that would result in less predictable behavior.

```
LATIN-1 +   UTF-8 =   UTF-8
  UTF-8 + LATIN-1 =   UTF-8
```

Combining a RAW string with UTF-8 is potentially unsafe. For the sake of predictable behavior, this will be considered a run-time error. The correct behavior would be to convert the UTF-8 string to a RAW string first.

```
    RAW +   UTF-8 = <ABORT>
  UTF-8 +     RAW = <ABORT>
```

# Prop Storage #

Regardless of implementation, it will necessary to be able to track the type of string being stored in a prop for sanity's sake.

Currently there is a single prop flag used for tracking strings, `PROP_STRTYPE` (defined in `inc/props.h`). To avoid needless code duplication, it's probably better to avoid defining a new prop types entirely and instead use any new flags to denote additional characteristics of the existing `PROP_STRTYPE`.

There _should_ be some sort of visual way of distinguishing these string types from each other when examining a propdir. (probably by adjusting the `(str)` readout slightly)