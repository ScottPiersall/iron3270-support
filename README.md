# Iron 3270 support

Bug reports, questions and feature requests for **Iron 3270**, a TN3270 terminal for Visual Studio
Code and other VS Code-based editors. It connects to z/OS, z/VM and other IBM mainframe systems.

- **Install:** [Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=ScottPiersall.iron3270)
  · [Open VSX](https://open-vsx.org/extension/ScottPiersall/iron3270) (IBM Bob, VSCodium and others)
- **Website and user guide:** https://iron3270.scottpiersall.com/
- **Changes in each version:** the Changelog tab on the Marketplace page

This repository holds no source code. It's the public place to report problems and to follow how
they are fixed.

## Report a problem

[Open an issue](https://github.com/ScottPiersall/iron3270-support/issues/new/choose) and choose
**Bug report**. It helps to include:

1. **Version and platform:** the Iron 3270 version (from the Extensions view), the editor and its
   version (VS Code, IBM Bob, VSCodium and so on), and your operating system.
2. **What you did and what happened:** the exact error message, if there was one.
3. **The diagnostics log:** open the Command Palette and run **Iron 3270: Show Diagnostics Log**.
   It never contains screen data or keystrokes, but it does contain host names and ports.

## Keep private details out of issues

Issues are public. Before you post, remove:

- **Host names, IP addresses and ports** of your systems. Replace them with something like
  `myhost:992`.
- **User IDs**, and anything else that identifies you or your organization.
- **Passwords**, **certificates** and **private keys**: never post these, even partly.
- In screenshots: host names, user IDs, data on the screen, and your avatar.

Don't attach protocol trace files (`iron3270.logging.protocolTrace`). They contain everything sent to
and from the host, including passwords typed into fields. If a trace is needed, you'll be asked to
share it privately.

## Security problems

Don't report security problems in a public issue. See [SECURITY.md](SECURITY.md).

## Feature requests

Choose **Feature request** when you open an issue. Describe what you're trying to do on the host,
not only the feature you have in mind. That makes it easier to find the best solution.
