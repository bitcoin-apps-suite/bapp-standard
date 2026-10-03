# The bApp standard

**Version 0.1, draft.** The Bitcoin Corporation Ltd, October 2026.

This document says what a web app must do to be a **bApp**: an app that runs inside bWallet and uses the wallet's identity, `$handle` and payments instead of building its own.

Anything marked **Proposed** is not in bWallet yet. Everything else describes how bWallet behaves today (source: `yours-mobile-bcorp`, branch `bwallet`). The key words MUST, SHOULD and MAY are used as in RFC 2119.

## 0. Naming

- A bApp's name starts with a lowercase **b** followed by a capital letter: bChat, bMail, bDrive.
- A bApp SHOULD have its own BSV-21 token, ticker `$b<Name>` (for example `$bMail`), declared in the manifest's `token` field.
- A lowercase b after `$` marks a bApp. A ticker without it (`$DIVVY`, `$MONEYBUTTON`) is a standalone venture, not a bApp. Ventures can still run in bWallet's browser, but they aren't listed under bApps.

## 1. Detection

bWallet's in-app browser adds markers to the user agent:

| Platform | User agent suffix |
|----------|-------------------|
| iOS (WKWebView `applicationNameForUserAgent`) | `Mobile/15E148 bWallet/1 YoursWalletMobile/1` |
| Android (WebView `setUserAgentString`) | `<default UA> bWallet/1 YoursWalletMobile/1` |

A bApp detects the wallet like this:

```js
const inWallet = /\bYoursWalletMobile\/\d+/.test(navigator.userAgent);
const inBWallet = /\bbWallet\/\d+/.test(navigator.userAgent);
```

- `YoursWalletMobile/1` is the compatibility marker. Match it to support any Yours-derived mobile wallet.
- `bWallet/1` marks bWallet specifically. The number is the bridge version; it changes only when the bridge breaks compatibility.
- Detection is a hint for layout. It is not a security boundary, because any browser can fake a user agent. For anything that matters, call the provider (section 3).

When the app isn't inside the wallet, it SHOULD work as a normal website or show an "Open in bWallet" prompt.

## 2. Layout contract

### 2.1 Today

The browser opens the bApp in a full-screen native WebView. On iOS a 48pt native bar (back, page title/URL, close) sits under the top safe area. On Android the WebView is padded by the system-bar and keyboard insets. The page gets the whole area below the native bar.

### 2.2 Target: inside the wallet frame (**Proposed**)

A bApp is rendered in the viewport between bWallet's two bars:

| Bar | Size (in the wallet source) |
|-----|------------------------------|
| Top navbar | `h-14` (3.5rem, 56px) |
| Bottom tab bar | `3.75rem` (60px) |

Rules (they apply today as well, so the move to the frame doesn't break anything):

1. **No navigation of your own.** Don't add a top app bar, a bottom tab bar, a hamburger menu or a "back to home" header. The wallet provides them.
2. **No fixed elements at the edges.** If you need a sticky action bar, place it inside your scroll container, not with `position: fixed; bottom: 0`.
3. **Safe areas.** Use `viewport-fit=cover` and pad with `env(safe-area-inset-*)`. Inside the frame those insets resolve to 0, because the wallet bars already handle them.
4. **Height.** Use `100dvh` or `height: 100%` on `html, body`. Don't hard-code 100vh.
5. **No zoom traps.** Inputs use a font size of at least 16px, so iOS doesn't zoom on focus.

### 2.3 Theme tokens

bWallet is dark only. bApps SHOULD match it:

```css
:root {
  --bapp-bg: #010101;    /* wallet background (the native browser also uses rgb(1,1,1)) */
  --bapp-gold: #FFD24D;  /* accent: primary buttons, the b mark, links */
  --bapp-ink: #F4F1E8;   /* body text */
  --bapp-muted: #A9A69C; /* secondary text */
  --bapp-edge: #23252B;  /* borders, dividers */
}
html { background: var(--bapp-bg); color-scheme: dark; }
```

Only `--bapp-bg` and `--bapp-gold` are required. The others are recommended values taken from the bWallet site.

## 3. Identity

### 3.1 The provider

bWallet injects its provider at document start, into the main frame only, in every page the in-app browser opens. The page gets **`window.CWI`**: a BRC-100 `WalletInterface` (from `@1sat/wallet-browser`). Requests go over a native bridge (`webkit.messageHandlers.yours` on iOS, `yoursNative` on Android). The wallet stamps each request with the page's real origin; the page can't set it.

The methods bWallet answers are the BRC-100 set:

`isAuthenticated`, `waitForAuthentication`, `getVersion`, `getNetwork`, `getHeight`, `getHeaderForHeight`, `getPublicKey`, `createSignature`, `verifySignature`, `createHmac`, `verifyHmac`, `encrypt`, `decrypt`, `createAction`, `signAction`, `abortAction`, `internalizeAction`, `listActions`, `listOutputs`, `relinquishOutput`, `revealCounterpartyKeyLinkage`, `revealSpecificKeyLinkage`, `acquireCertificate`, `listCertificates`, `proveCertificate`, `relinquishCertificate`, `discoverByIdentityKey`, `discoverByAttributes`.

Only `isAuthenticated` and `getVersion` work without the user's permission. Every other call goes through bWallet's permission prompts, scoped to the page's origin.

> The legacy Yours provider (`window.yours` with `connect`, `getAddresses`, `getSocialProfile`, `sendBsv`, `signMessage`) is **not** injected by bWallet's mobile browser. bApps MUST use `window.CWI`.

### 3.2 Connect

"Connect" means getting the user's identity key:

```js
const wallet = window.CWI;
const { authenticated } = await wallet.isAuthenticated({});
if (!authenticated) await wallet.waitForAuthentication({});
const { publicKey } = await wallet.getPublicKey({ identityKey: true });
```

The **identity key** (a 33-byte compressed public key, hex) is the user's account ID in your app. Store it instead of an email or a username.

### 3.3 No passwords

A bApp MUST NOT ask a bWallet user for a password, email login or seed phrase. For a server session, use challenge and response:

1. The server sends a random nonce.
2. The app calls `createSignature` with the nonce as `data`, a protocol ID such as `[2, 'bapp login']` and a `keyID`.
3. The server checks the signature against the identity key (or the app calls `verifySignature`) and opens a session.

### 3.4 The `$handle`

bWallet users have a `$handle`, which is also their paymail: `alias@bwallet.space`. The user claims the alias in bWallet (Settings › Identity), where it is registered against their identity key on the wallet's paymail server (`pay.bwallet.space`; client in `src/mobile/names/paymail.ts`). The mapping from identity key to paymail is public.

A bApp resolves the handle from the identity key it got in 3.2:

```
GET https://pay.bwallet.space/api/paymail/lookup?key=<identityKey>
→ 200 { "paymail": "alice@bwallet.space" }      the user's handle is $alice
→ 404                                            no handle claimed yet
```

Rules:

- Show `$alias`, the paymail's local part. Fall back to the shortened identity key when the lookup returns nothing: not everyone claims a handle.
- Never ask the user to type their handle, and never treat a typed handle as identity. The identity key is the account id; the handle is a label for it, and it can be renamed or deleted.
- Call the lookup from your server or proxy it through your origin. The paymail server's CORS policy is not part of this spec.
- `GET https://pay.bwallet.space/api/paymail/id/<paymail>` is the paymail's public profile (bsvalias). A `404` means the alias is free. When the profile publishes an ordinals address, send the user's tokens and ordinals there.
- A HandCash `$handle` is a different namespace (`alias@handcash.io`). Only a `bwallet.space` paymail is a bWallet handle.

**Proposed:** a provider method `getHandle()` → `{ handle: '$alice', paymail: 'alice@bwallet.space', identityKey }`, gated by the same permission prompt as `getPublicKey`, so in-frame bApps need no network call. Until it ships, the lookup above is the way.

## 4. Payments

### 4.1 The rule

**Charge in dollars. Users pay in sats.** A bApp quotes every price in USD. The wallet converts at the live BSV/USD rate when the user pays, and shows the dollar amount first with the sats alongside (for example "$0.01 (2,174 sats)").

### 4.2 Today

Payments are BRC-100 `createAction` calls with outputs in satoshis. A bApp converts USD to sats itself at a current rate and states the USD amount in the action `description`. bWallet shows its confirmation prompt.

**One-click pay** (Settings): when it's on, bWallet skips the confirmation for a small paying action if all of these hold:

- the amount is a whole number of sats at or under the user's per-action limit (default 1,000 sats), and
- fewer than 5 actions were auto-approved in the last minute, and
- the auto-approved total in that minute stays under 5 × the limit.

Anything else gets the normal confirmation. The limit is never exceeded. (Today one-click applies to the wallet's own actions such as tips and locks; extending it to bApp requests is **Proposed**.)

### 4.3 USD payment requests (**Proposed**)

```js
await wallet.requestPayment({
  usd: '0.01',
  to: 'paymail-or-identity-key',
  memo: 'Tip for post #42',
});
// → { txid, sats, rate }
```

The wallet converts at the live rate, applies one-click limits (in USD terms), and returns the sats actually paid and the rate used. Until this ships, use `createAction` as in 4.2.

## 5. Back navigation

- bWallet owns Back. bApps MUST NOT intercept the Android back button or draw their own back arrow for leaving the app.
- Today, in the dApp browser, Back goes through the WebView history (`history.back()`) and closes the browser when the history is empty. On iOS, the native bar's chevron and the edge swipe do the same. So a bApp's in-app screens SHOULD be real history entries (`history.pushState` or real URLs).
- Inside the wallet's own UI, sheets and modals register with a back registry (`useBackClose(open, onClose)` in `src/mobile/backStack.ts`): Back closes the most recently opened sheet first. **Proposed:** expose the same thing to bApps, `wallet.onBack(handler)`, so a bApp's open sheet closes before the page goes back.

## 6. Manifest: `bapp.json` (**Proposed**)

Every bApp serves a manifest at `/bapp.json` on its origin. Schema: [`schema/bapp.schema.json`](schema/bapp.schema.json).

| Field | Required | Meaning |
|-------|----------|---------|
| `bapp` | yes | Spec version, `"0.1"` |
| `name` | yes | Display name, `b` + capital, e.g. `"bChat"` |
| `short_name` | yes | Up to 12 characters, for the icon label |
| `icon` | yes | Square PNG or SVG, at least 192px, HTTPS or relative to the origin |
| `start_url` | yes | Where the wallet opens the app |
| `category` | yes | `work`, `media`, `social` or `money` (mirrors the Apps tab groups) |
| `permissions` | yes | What the app will ask for: `identity`, `handle`, `sign`, `encrypt`, `pay`, `tokens`, `certificates` |
| `price` | yes | `{ "model": "free" \| "per_use" \| "subscription" \| "tips", "usd"?: "0.01", "period"?: "month" }`, always in USD |
| `token` | recommended | `{ "ticker": "$bMail", "tokenId": "<txid>_<vout>" }`, the app's BSV-21 token |
| `developer` | yes | `{ "name", "identityKey" }`, the developer's identity key (33-byte compressed, hex) |
| `source` | no | Repo URL if the app is open source |
| `signature` | yes | Signature by `developer.identityKey` over the manifest |

### 6.1 Signature

1. Take the manifest without `signature`.
2. Serialise it canonically (JSON with keys sorted, no whitespace, UTF-8).
3. Sign the bytes with the developer's identity key (ECDSA over SHA-256, the same scheme `createSignature` uses with `protocolID: [2, 'bapp manifest']`, `keyID: '1'`, `counterparty: 'anyone'`).
4. Put the DER signature, as hex, in `signature`.

### 6.2 Verified tick

bWallet already has verified issuers: every token it mints carries an issuer statement (Bitcoin Schema MAP + AIP, in the same transaction) signed by the minter's identity key. The wallet checks that key against bit-sign's issuer registry (handle + KYC) and shows a tick when the issuer is `verified`. A bApp listing uses the same key. If `developer.identityKey` is a verified issuer and the signature checks, the listing gets the tick. If the token in `token` was issued by the same key, the token's tick and the app's tick match.

## 7. Listing under bApps

The Apps tab in bWallet has three pages: **Home**, **bApps** and **Other apps**. Today the bApps page is a curated list compiled into the wallet (`src/mobile/bapps.ts`): `name`, `url`, a one-line `verb`, a `group` (Featured, Work, Media, Social & money), `status` (`live` or `demo`), an `icon` and an optional `source` repo. Demo apps are labelled Demo.

**Proposed:**

1. A developer submits the origin. bWallet fetches `/bapp.json`, checks it against the schema and verifies the signature.
2. Only names that follow section 0 are listed under bApps. Everything else goes under Other apps.
3. The listing shows the icon, name, a one-line description, the USD price and the tick (6.2). If `token` is set, it links to the token's market and its room.
4. The listing updates when the manifest changes and the new signature verifies.

## 8. Security

- **Origin checks.** The wallet takes the origin from the native WebView, not from the page, and refuses anything that isn't `http(s)`. Permissions are kept per origin. Only `https` origins can be listed as bApps (**Proposed**).
- **No key access.** A bApp never sees private keys, seed phrases or the wallet's storage. It gets public keys, signatures, ciphertext and transactions, and only through the provider and its prompts.
- **Main frame only.** The provider is injected into the top frame. Iframes inside a bApp don't get `window.CWI`.
- **No impersonation.** A bApp MUST NOT draw UI that looks like a bWallet prompt, a balance or a seed entry form.
- **Manifest integrity.** A listing whose signature fails, or whose identity key changes, loses the tick until it's re-verified (**Proposed**).

## 9. Versioning

- This spec uses `MAJOR.MINOR`. 0.x drafts may change anything.
- The manifest declares the version it targets in `bapp`.
- The user agent marker (`bWallet/1`, `YoursWalletMobile/1`) is the bridge version. It's independent of the spec version and changes only when the bridge breaks compatibility.
- New provider methods are feature-detected (`typeof wallet.requestPayment === 'function'`), not inferred from version numbers.

## 10. Open questions

1. Handles ship as paymail aliases resolvable from the identity key (3.4). Should the wallet also return them in-band: a `getHandle()` method, or a BRC-52 certificate through `proveCertificate`?
2. Can a bApp ask for a one-click allowance of its own (for example "$1 a day for this app"), separate from the wallet-wide limit?
3. Which price oracle does the wallet use for `requestPayment`, and how long does a quote stay valid?
4. Canonical JSON: adopt RFC 8785 (JCS) exactly, or a simpler sorted-keys rule?
5. Do listings need on-chain anchoring (a manifest hash in an OP_RETURN), or is a signed file on the origin enough?
6. Should `bWallet/1` include the wallet's app version for debugging?
7. How should a bApp that is also a desktop or extension app behave when the Yours extension is present?
8. Who can list: anyone who signs, or only verified issuers?
