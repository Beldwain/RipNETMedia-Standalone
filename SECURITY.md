# Security Policy

## Supported Versions

RipNETMedia Standalone is distributed as a Windows installer from this
repository's [Releases](../../releases). Security fixes are published in a new
release; we support only the **latest** version. Please update before reporting
an issue, as it may already be fixed.

| Version | Supported          |
| ------- | ------------------ |
| 1.0.x (latest) | :white_check_mark: |
| < 1.0   | :x:                |

You can see your installed version in the app under **About**.

## How RipNETMedia is designed (scope)

A little context on what is and isn't in scope:

- The app runs a local background service and a dashboard that listen **only on
  `127.0.0.1` (your own machine)** — there is no network login and the service
  is not meant to be reachable from other computers. Exposing it to a network
  (for example by port-forwarding or a reverse proxy) is unsupported and outside
  this policy.
- The installer and app are **not yet code-signed**, so Windows SmartScreen will
  warn on first launch. That is expected and is **not** a vulnerability — see the
  [Installing guide](/../../wiki/Standalone‐Installing).
- The **diagnostics** bundle (**About → Download diagnostics**) intentionally
  **removes API keys** before packaging. If you ever find a secret leaking into
  that bundle, into logs, or into exported settings where it shouldn't, please
  report it.

In scope: anything that lets another program or user on the machine (or the
network) do something they shouldn't — read your secrets, run code, tamper with
rips or the installer, or reach the service from off-box.

## Reporting a Vulnerability

**Please do not open a public issue for security problems.** Report privately
through **GitHub private reporting**: on this repository, go to the **Security**
tab → **Report a vulnerability**. This opens a private advisory that only you and
the maintainers can see.

Please include, where you can:

- The app **version** (from the About page) and your **Windows version**.
- What you found and **how to reproduce it**, step by step.
- The impact you think it has.
- Any relevant diagnostics or logs. **Double-check they contain no passwords or
  keys** before sending.

### What to expect

- **Acknowledgement** within **7 business days**.
- An initial assessment (accepted / need-more-info / declined, with reasoning)
  within **14 business days**.
- If accepted, we'll keep you updated on progress and aim to ship a fix in a new
  release as quickly as the severity warrants, then publish an advisory.
- If declined, we'll explain why (for example, out of scope as described above).

We practice **coordinated disclosure**: please give us a reasonable chance to
release a fix before sharing details publicly. With your permission, we're happy
to **credit you** in the release notes and advisory.

Thank you for helping keep RipNETMedia users safe.
