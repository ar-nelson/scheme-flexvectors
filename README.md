# Flexvectors (SRFI 214)

> This is an outdated repository. The canonical repository for SRFI 214 is
> https://github.com/scheme-requests-for-implementation/srfi-214

An implementation of flexvectors (also known as dynamic arrays or arraylists) in
R7RS Scheme. Contains an R7RS library (`flexvectors.sld`) and a test suite
(`tests.scm`).

This implementation supports Gauche, Sagittarius, and Chibi. If all three of
these are installed (binaries `gosh`, `sash`, and `chibi-scheme`), running
`make` should run the test suite for all three. To run the test suite in
a specific Scheme, use `make test-gauche`, `make test-sagittarius`, or `make
test-chibi`.
