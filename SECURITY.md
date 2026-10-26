# Security Policy

This repository holds the Cashu Card client library: the host half that
blinds, talks to the mint and verifies what the card signs. A bug here can lose
the money on a card, so reports are welcome and taken seriously.

## Reporting a Vulnerability

Report it privately through this repository's security advisories:
<https://github.com/cashucard/client/security/advisories/new> (the **Security** tab,
then **Report a vulnerability**). That makes a private advisory that only the
maintainers can see. You can also email
[security@cashucard.org](mailto:security@cashucard.org).

Please do **not** open a public GitHub issue, pull request or discussion for a
vulnerability.

Include as much of the following as you can:

- A description of the vulnerability and its potential impact
- Steps to reproduce (proof of concept, affected version or commit)
- Any suggested remediation

## What to Expect

- Every report is read, and we will keep you informed as we investigate and
  remediate.
- Please allow us a reasonable window to fix the issue before public disclosure.

## Scope

This policy covers this repository: the library in `src/`, its tests and the
documents. The applet, the provisioner, the ops service and cashucard.org are in
[cashucard/cashucard](https://github.com/cashucard/cashucard), and the protocol
is specified in [cashucard/nut](https://github.com/cashucard/nut); a private
report here reaches the same maintainers. Wallet and till apps that use this
library have their own policies.
