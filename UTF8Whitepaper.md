# Introduction #

Unicode has been around awhile, but a practical implementation that is useful to players is an elaborate set of challenges.

This document is broken up into documentation for servers and clients, prefaced with some details applicable to both implementations.

# Reserved Characters #
The recommendations below are derived from Unicode.org's [Private-Use Characters, Noncharacters & Sentinels FAQ](http://www.unicode.org/faq/private_use.html).

## Interpretation of Private Use Characters ##
The interpretation of Private Use Characters should be defined by the administrators of a game, and no limitations on their interchange should be imposed by client or server software. An unmodified client will not render these characters in any useful way (most clients will see the replacement character, �). That said, if a Star Trek themed game is interested in implementing one of the Klingon alphabets, there is no reason to stand in the way of custom clients or client plugins being able to implement this functionality.

## Interpretation of Noncharacters ##
This whitepaper recommends that Unicode noncharacters be classified as _internal to the application_, whether that application be a client or a server. The meaning of noncharacters is defined by the respective software maintainers and _must not_ be exchanged in the game's primary user I/O loop.

Noncharacters can be used for any internal purpose that requires dedicated characters that will never be accepted on the wire. The most common use would be internal escape characters that are not defined by US-ASCII or Unicode. In this fashion noncharacters are similar to how the 8th bit of ASCII could be utilized in the early days of computing (history lesson: [8-bit cleanliness](https://en.wikipedia.org/wiki/8-bit_clean)), with the following caveats:

  * Noncharacters within the Unicode standards are protected, and will never be reclassified. (unlike ASCII's 8th bit)
  * Noncharacters encode at least three characters, and are not as space efficient.

Clients _should not_ accept noncharacters as user input. Games should discard these characters from user input if they are encountered.

Games _must_ treat these characters as "zero width" in a character context, and as their length in UTF-8 bytes in a byte context.

Games _may_ define a subset of noncharacters that can be manipulated by softcode and/or the game administrators, but there should _never_ be an expectation that those characters will be received as input from a client.

Clients and servers _may_ include noncharacters if it is not direct I/O to the user. Examples of this include writing noncharacters out to disk, or transmitting them over the network in a JSON encoded format that is not rendered directly to a user's terminal.

# The Server #
The basic rules are pretty well-known. The server itself must implement a Unicode aware encoding, and translate input from all descriptors into a Unicode format. This can be wchar\_t strings, UTF-8 encoded C strings, whatever floats your boat.

## Supported Encodings ##

Each connection to the game should have its own configured encoding. A given player may have multiple active sessions to the game, each with their own separate encodings. The game must assume a safe default for connections that offer no hints as to their encoding, but provide a fallback command for adjusting the active encoding in the event that the player wishes to adjust this. Persisting this preference across logins is optional, and probably not a good idea.

The first thing you need to decide is what encodings you intend to allow as input. I recommend that you keep this small, and avoid deviating from the following list. I cover individual bytes that should be discarded below, but I do not go into the subject of control characters 0-31 + 127 (you are assumed to know this), or invalid UTF-8 sequencing (which will be covered later).


  * [ISO-8859](https://en.wikipedia.org/wiki/ISO/IEC_8859-1)
AKA Latin-1. Characters above point 127 are selectively interpreted as Latin characters. It should be noted that the ÿ character at code point 255 is the same byte used to represent Telnet IAC; special considerations involving the translation of this character will be covered later.

Invalid bytes: 127-159

IAC+IAC handling: ÿ

This is the most common interpretation of characters 128-255 by clients that have not negotiated a state with the server.

  * US-ASCII
AKA ASCII. This should be considered the "safe" encoding; characters above code point 127 are garbage outside of IAC escape sequences.

Invalid bytes: 127-255

IAC+IAC handling: garbage

  * [UTF-8](https://en.wikipedia.org/wiki/UTF-8)
This is what most players and game authors are thinking about when they discuss Unicode support. UTF-8 is backwards compatible with US-ASCII for code points 0-127, and uses special sequences of bytes 128-244 to represent other character sets.

Invalid bytes: 192-193, 245-255, invalid multi-byte sequences

IAC+IAC handling: garbage

  * Optional: [IBM478](https://en.wikipedia.org/wiki/Code_page_437)

AKA IBM Code Page 478, AKA CP478, AKA OEM478. This character set is popular within the MUD community for the purpose of art creation, but is unlikely to be seen outside of those circles. Use your discretion when choosing to implement this, as supporting it adds additional complexity to the tasks ahead that are otherwise avoidable.

It should be noted that unlike all of the character sets above, bytes 1-31 are not reserved for control codes and can be used at

Invalid bytes: None

IAC+IAC handling: garbage

## Default Encoding ##
Most clients connecting to your game will not negotiate an encoding. For these you must assume a default: either ISO-8859-1 or US-ASCII. It's best to leave this configurable for game administrators, as the best default for the situation requires knowing their individual audiences and whether the majority of their users will render bytes 127-254 as Latin or garbage.

## Negotiating Encoding ##
There are two recommended methods. The UTF-8 feature advertised by these features is usually a user preference: the client may support UTF-8 and have a useful font selected, or the user may have no idea what they're doing. There is nothing that can be done about this. If the client has asked for UTF-8 and you are sending a valid byte stream, it's not your problem to deal with.

  * [MTTS](http://tintin.sourceforge.net/mtts/)
This method is preferred over the TELNET CHARSET option due to its simplicity, and the fact that it can also negotiate support for 256 colors at welcome screen time. Two desirable features for the price of one.

The only downside is that this is not backed by a RFC. That said, neither is 256 color support, or numerous other MUD extensions.

  * [TELNET CHARSET](http://tools.ietf.org/html/rfc2066)
This method is endorsed by an RFC with an Experimental status. It's not an accepted Standard, but it's the closest you're going to see to one. This takes more work to implement, and not many clients support this. Your call.

## Font Considerations ##
Before we get into the subject of how to filter your input and output streams, it's important to understand that the usefulness of Unicode is largely constrained by the font support in your userbase. It is only as good as your least common denominator font. Outside of shipping a custom client for your game (which not everyone will use), or endorsing an official font for the game (which most people will ignore), this is nothing you have control over.

Most of the recommendations in this document build on the character remappings defined by the FANSI standard, with some additional considerations based on the assumption that most Unicode aware fonts will implement [WGL4](https://en.wikipedia.org/wiki/Windows_Glyph_List_4) characters at a bare minimum.

## Universal Markup ##
The choice of whether to render a question mark for a byte that cannot be translated or dropped entirely is a debated one. This author is of the particular mindset that ordinary users do not get much value from seeing streams of question marks rendered to their screen. Not only this, there needs to be a way for users on more limited encodings to interact with shared data fields such as room descriptions. In the interests of achieving both goals, the following "contexts" are defined:

**Examine Context**: Any context where the user is examining a field of information vs. having it rendered to them. This is most commonly seen by players when they are using the examine command. If your codebase implements an inline language like MUSHCode or MPI, any context where these characters are rendered and not processed is considered the Examine Context. You can also think of this as the "Markup Context".

**Rendering Context**: Any context outside of an Examine Context. Think of this as viewing the contents of a webpage vs. viewing the source code.

It is recommended to never render question marks in a Rendering Context. These are not of much use to third party observers, and are somewhat jarring in collaborative roleplay environments.

In an Examine Context, it is _strongly recommended_ to not render question marks but instead output the characters in a Universal Character Name (UCN) escaped format, similar to how they can be legally used within the source code of C (C99 standard) and C++. These take the form of a `\u` sequence followed by a four digit number (for the Basic Plane), or a `\U` sequence followed by an eight digit number (for the Supplementary Planes). Support for rendering the Basic Plane using the `\U` syntax should be implemented, but its use by users should be discouraged.

Example: ☺ (3 bytes: e2 98 ba) cannot be rendered to a US-ASCII or ISO-8859-1 descriptor, but can be safely rendered as `\u263A` or `\U0010263A` (overkill).

Implementing support for the UCN escaped format on all descriptors allows for the more limited codings to copy text rendered in an Examine Context and paste them back into the game without alteration, without destroying the formatting of the extended characters in the process.

It is acknowledged that this goal can also be achieved through implementation specific mechanisms that codebases choose to define on their own, but this method is proposed as a neutral standard that can be more widely adopted.

## Output Filtering ##
When rendering output to descriptors, your goals are as follows:

  * Only render bytes that are legal for the target encoding. No exceptions. Output garbage is the hallmark of a sloppy implementation and works against the support of your userbase.
  * If it's possible to translate a character literally from one encoding to another, always do so. This mostly concerns translation between ISO-8859-1 and UTF-8; US-ASCII is stuck dropping most of it.
  * If it's possible to translate a character to something that would still be fairly representative in the target encoding, do so using the Translation Tables at the end of this document.
  * Render a UCN escaped character, drop the byte, or convert it to a question mark (?).

Unicode has some additional considerations, such as illegal multi-byte sequences, [its own control characters](https://en.wikipedia.org/wiki/Unicode_control_characters) and Private Use Planes that can only be used to render characters that are implementation specific. These will be elaborated on in a future revision of this document, but for now this author needs to get to a point where he can feel comfortable clicking on Save. Any characters which are disruptive or potentially unsafe should not be rendered unless it is certain that they are not sourced from normal player input. (similar to illegal bytes outside of IAC sequences)


## Input Filtering ##
Processing input is a slightly easier affair:

  * If the byte is not legal for the negotiated character set and not part of an IAC escape sequence, it should be discarded as garbage.
  * In addition to any special escaping that your codebase currently implements, it is strongly recommended to accept UCN escaped characters as described in previous sections.
  * Allow Private Use Characters. See "Reserved Characters".
  * Forbid Noncharacters. See "Reserved Characters".



# The Client #
For another day.


# Character Conversion Tables #
Also for another day. These will be largely the same as the FANSI 2.0 conversion tables, but with some additional recommendations specific to rendering Unicode characters in simpler character sets (quoting, etc.), as well as addressing some extremely common problems in supported glyphs (fonts supporting only the WGL4 glyphs present in Unicode's [box drawing characters](https://en.wikipedia.org/wiki/Box-drawing_character).