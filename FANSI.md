# Intro #

This document outlines ZetaMUCK's compliance status with the [FANSI Specification](http://www.fansi.org/Specification.aspx).

ZetaMUCK is unlikely to ever be fully FANSI compliant. Our desired approach is to use the parts of the specification that add the most value to the user, and gloss over any parts that would require coding time to solve challenges that don't really apply to us.

# Current Status #

**Planning.** This document covers the intended future design, but work has not yet started.

# What is FANSI? #

`FANSI 2.0 is an enhanced ANSI Art specification for MUDs (multi-user dungeons) that allows 256 colors and 151 extended characters. A freeware mouse-driven drawing application, MUSHii, is also available to assist with the creation of FANSI art.`

(taken from http://www.fansi.org/Index.aspx)

# Charsets #

ZetaMUCK will implement the following character sets:

  * US-ASCII (classic 7-bit)
  * BINARY (file and socket I/O)
  * ISO-8859-1 (Latin-1, excluding character 255)
  * UTF-8

ZetaMUCK will not implement the following character set:
  * CP437 (IBM Code Page 437)

Given the state of Unicode compatibility in modern MU clients and the number of users expected to leverage CP437, adding this support is not perceived to be a good investment of time.

# Color Conversion #

ZetaMUCK will implement the following color conversions:

  * [256 → 16](http://www.fansi.org/256Colors.aspx)

ZetaMUCK will not implement the following color conversions:
  * 256 → MXP
  * MXP → 256 (not a part of the standard anyway)

The only MXP client commonly used with ZetaMUCK (and ProtoMUCK) is MUSHclient, which supports the 256 color scheme in addition to MXP. While we may implement support for MXP at a later date, MXP without 256 color support is an extremely unlikely scenario given that it takes MU client authors more work to implement MXP.

# Character Conversion #

ZetaMUCK will implement the following character conversions:

  * ISO-8859-1 → UTF-8
  * ISO-8859-1 → US-ASCII
    * ISO-8859-1 characters not present in the US-ASCII table will be downmixed.
  * UTF-8 → US-ASCII.
    * UTF-8 characters corresponding to ISO-8859-1 or CP437 entities will be downmixed.
  * UTF-8 → ISO-8859-1
    * In cases where direct character mappings exist, those will be used.
    * UTF-8 characters corresponding to CP437 entities will be downmixed.

All downmixing will follow the guidelines set forth by the following tables of the [FANSI Specification](http://www.fansi.org/Specification.aspx), in order of preference:
  * **IBM/OEM To ISO 8859-1 And/Or ASCII Conversion Table**
  * **IBM/OEM To Standard ASCII Conversion Table**