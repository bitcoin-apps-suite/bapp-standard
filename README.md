# bitcoinappsuite: the bApp standard

A **bApp** is a web app built to run inside **bWallet**, The Bitcoin Corporation Ltd's mobile BSV wallet (a Capacitor fork of [Yours Wallet](https://github.com/yours-org/yours-wallet)).

The idea is simple. A bApp lives in bWallet's frame, between the wallet's top bar and its bottom tab bar. It brings no navigation of its own: no app chrome, no login screen, no wallet picker. It plugs into what the wallet already has:

- **Identity.** The user's identity key and `$handle`. There are no passwords.
- **Payments.** The app quotes prices in US dollars and the wallet pays in sats at the live rate. The rule: *charge in dollars, users pay in sats.*
- **Back.** The wallet owns the Back gesture and the tab bar.

## Naming

- bApp names start with a lowercase **b**: bChat, bMail, bDrive.
- A bApp usually has its own BSV-21 token, `$b<Name>` (for example `$bMail`). The manifest's `token` field is optional but recommended. In the Apps tab, a listing links to the token's market and room.
- The convention is strict. A lowercase b after `$` marks a bApp. A ticker without the b (`$DIVVY`, `$MONEYBUTTON`) is a standalone venture, not a bApp.

## What's here

| Path | What it is |
|------|------------|
| [SPEC.md](SPEC.md) | The standard, v0.1 draft. Parts that bWallet doesn't support yet are marked **Proposed**. |
| [schema/bapp.schema.json](schema/bapp.schema.json) | JSON Schema for the `bapp.json` manifest |
| [examples/hello-bapp/](examples/hello-bapp/) | A minimal static bApp: detection, connect, and a $0.01 tip |

## Status

v0.1 is a draft. bWallet already supports detection, the injected BRC-100 provider (`window.CWI`), in-frame hosting over the XDM substrate, dark-theme hosting, the bApps listing, `$handle` resolution from the identity key (paymail aliases at `bwallet.space`, SPEC section 3.4) and one-click limits. The manifest, signed listings, an in-band `getHandle()` and USD-denominated payment requests are **Proposed**.

Reference implementation: [bMint](https://github.com/b0ase/bcorp-mint) (`shared/lib/bwallet.ts`, `website/app/api/auth/bwallet/`) does detection, connect, handle resolution, challenge/response login and `createAction` inscriptions against this spec.

## License

[Open BSV License version 5](LICENSE), © 2026 The Bitcoin Corporation Ltd.
