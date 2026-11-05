# Security

Please report security problems in Iron 3270 privately, not in a public issue.

Email **spiersall@gmail.com** with "Iron 3270 security" in the subject. Include:

- the Iron 3270 version and your platform;
- what the problem is, and how to reproduce it;
- what an attacker could do with it.

Don't include your own passwords, private keys or other secrets. If a host is involved, describe it
rather than naming it.

You'll get a reply within a few days. Please give the problem a chance to be fixed before you
disclose it publicly. Fixes are released as a new version and noted in the changelog.

Examples of what counts as a security problem:

- TLS that accepts a certificate it should reject, or connects without TLS when TLS was configured.
- Passwords or other secrets written to logs, settings, or anywhere outside VS Code's secret storage.
- A way for host data or another extension to run commands or code through Iron 3270.
