# Security Policy

This repository holds BitBang's design documents and protocol writeups. It contains no
running code, so vulnerabilities generally belong in one of the implementation repos:

| Where the issue is | Report here |
| --- | --- |
| Command-line client, device-side listener, shell / file share / HTTP proxy | [bitbang-cli](https://github.com/richlegrand/bitbang-cli/security/advisories/new) |
| Signaling server, browser runtime (`bootstrap.js`, `sw.js`) | [bitbang-server](https://github.com/richlegrand/bitbang-server/security/advisories/new) |
| Python implementation | [bitbang-python](https://github.com/richlegrand/bitbang-python/security/advisories/new) |

If you are not sure which applies, pick any one of them. We would much rather sort it out
ourselves than have you not report it.

## What does belong here

Reports against the **design** rather than an implementation, which are genuinely valuable
and easy to overlook:

- A protocol-level weakness: a flaw in the signaling exchange, the key verification, or the
  identity and access model, that would remain after every implementation matched the spec
- A security claim in this repository that is **overstated or wrong**. The claims documents
  make specific promises about what the signaling server cannot do and what a network
  attacker cannot achieve. If one of those does not hold, that is a security finding, not a
  documentation nit, because people make deployment decisions on it.
- A gap between a document and what the implementations actually do, where the document is
  the more reassuring of the two

Report those privately at
[**bitbang-cli advisories**](https://github.com/richlegrand/bitbang-cli/security/advisories/new),
or email **security@bitba.ng**.

## What to expect

BitBang is maintained by a very small team, so an honest statement rather than a service
level agreement: we aim to acknowledge a report within a few days and to keep you updated
as we work through it. Complex issues take longer, and we will say so rather than go quiet.

We are glad to coordinate disclosure, agree an embargo date, and credit you. If you would
rather not be credited, say so.

## Safe harbor

We will not pursue or support legal action against anyone who makes a good-faith effort to
follow this policy: research on your own devices and accounts, no access to or modification
of other people's data, no degradation of service for others, and a reasonable window to
fix before public disclosure. If you are unsure whether something is in bounds, ask first.
