### Token Google Code shuttering statement. ###
We're aware of the fact that Google Code is closing its doors toward the end of the year and will be evaluating our options over the next few months. Issue tracking is a pretty important item for this project, so we'll be looking for a new home that allows us to import our history mostly intact.

Once a new home has been finalized, this page will be updated with links to the new site. Security fixes will continue to be published here in the meantime.


### We now return you to the regularly scheduled programming. ###


ZetaMUCK is an experimental fork of ProtoMUCK, which is in turn an experimental fork of FBMUCK. It has two primary goals:

  * Get the documentation and Unix build tools into a cleaner and more maintainable state.
  * Experiment with new ideas that may break compatibility with Windows.

All trunk commits to ZetaMUCK are profiled with the [Valgrind](http://valgrind.org/) tool in memcheck mode to ensure sanity. ZetaMUCK contains [numerous fixes](https://code.google.com/p/zetamuck/issues/list?can=1&q=Legacy%3DFBMUCK%2CProtoMUCK+-Status%3AInvalid+) for issues that have gone unnoticed in the upstream codebases.

New features include:
  * HTTP/1.1 connection persistence
  * Automatic `Content-Type: application/json` decoding in the webserver (requires JSON support to be compiled)
  * Serverside keepalives (TELNET IAC+NOP)
  * [MTTS](http://tintin.sourceforge.net/mtts/) / 256 color welcome screens