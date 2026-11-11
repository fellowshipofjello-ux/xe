# Web Wallet

> Client-side wallet using Web Crypto. Private keys never leave the browser.

Client-side wallet embedded in the `xe` binary, served as part of the embedded web UI (`xe node --ui` — wallet pages are on by default). Private keys never leave the browser. Zero JavaScript dependencies.

## Security [#security]

* Seeds encrypted with AES-GCM via Web Crypto API
* Keys derived via PBKDF2 (600,000 iterations, SHA-256)
* Fresh random IV per encryption
* Unlock lasts a hard 30 minutes from unlock (checked lazily on use) — there is no idle timer; lock explicitly or close the tab to end it sooner
* Seeds stored in localStorage as encrypted ciphertext only
* Private keys never leave the browser
* No JS dependencies — uses native Web Crypto

> **Warning:** after unlock, the raw passphrase is cached in `sessionStorage`
> under `xe.session.unlock` for up to 30 minutes so page navigations don't
> re-prompt. Any code running in the same origin can read it — roughly
> equivalent in risk to holding the decrypted seed in JS memory. The vault on
> disk stays encrypted regardless. Use the lock button on shared machines.

## Features [#features]

* **Multi-wallet management** — create, import, rename, delete wallets
* **Send/Receive** — transfer XE between accounts; pending sends are claimed with a per-item receive button (there is no auto-receive polling)
* **Faucet** — request testnet XE; the button appears when the node is started with a faucet target
* **P2P Chat** — real-time delivery, ed25519 signed envelopes
* **Provider dashboard** — read-only view of providers; creating leases and VM/SSH access are CLI-only
* **DAO governance** — draft, sign, submit state chain blocks
* **State inspector** — browse state chain blocks and KV entries
* **Client-side signing** — blocks are signed in the browser, but not offline: the node supplies the frontier, balance, network ID, and PoW parameters, and the same call submits the block

## Enabling [#enabling]

```bash
xe node --ui                     # embedded UI, wallet pages included (default)
xe node --ui --wallet=false      # UI without wallet (explorer only)
```

Wallet pages are enabled by default whenever the UI is on; `--wallet=false`
makes the node serve 404 for `/wallet/*`. Because keys stay client-side this is
a containment convenience, not the security boundary — the UI binds to
`--ui-bind` (default 127.0.0.1), so exposing it publicly is an explicit choice
that should go through a reverse proxy.

## Technology [#technology]

Plain HTML + ES modules + Web Crypto. Embedded in the Go binary via `//go:embed`. No build step, no framework, no bundler. Signing requires Web Crypto Ed25519: Chrome 137+, Firefox 129+, Safari 17+.

## Recovery phrases and backups [#recovery-phrases-and-backups]

Create shows 24 numbered English BIP39 words. Save them, then enter three randomly
chosen words before the wallet can be used. The phrase is your only backup unless
you separately keep the hex seed. Anyone with the backup controls the account.
Import accepts 12, 15, 18, 21 or 24 words, or a 64-hex signing seed.

Advanced controls expose the optional BIP39 passphrase and imported account
index. When creating a phrase, enter the passphrase twice. Creation stays blocked
until both entries match exactly; both fields are cleared after use. This also
applies when adding a wallet from manage-wallets. Import asks once; check the
restored address. The derivation is BIP39 + SLIP-0010 ed25519 at `m/44'/1'/account'`, default
account 0. Coin type 1 is provisional. Add wallet can create another phrase or
derive the next account from the current stored phrase. For a protected phrase,
re-enter its BIP39 passphrase. The wallet checks the passphrase against the current
account's decrypted signing seed and refuses a mismatch without changing the
vault. Leaving the passphrase option unticked checks an empty passphrase. The
input is cleared after success or a mismatch.

The BIP39 passphrase is separate from the vault password. It is never stored or
cached. Keep it separately and avoid a weak passphrase. On import, a wrong
passphrase opens a different, empty account. The existing sessionStorage unlock cache holds only
the vault password.

The Edge address generator also requires matching entries before Apply passphrase
is enabled. Applying clears both fields and resets the three-word check. New
address generates a new phrase at account 0.

Show backup decrypts the stored phrase and shows its account index. Hex is
available behind an advanced control. A wallet imported from hex has no phrase;
show backup explains this and displays its hex instead. Wallets made before v2
restore from their hex seed, because old #1005 words derive a different account.

The vault's `encrypted_seed` is unchanged. Optional `encrypted_phrase`,
`phrase_iv` and `account` fields are additive. Phrase encryption uses the same
PBKDF2-SHA256 vault key and a fresh AES-GCM IV. Existing vaults, DAO and chat open
without migration. The signing seed remains authoritative.

---

Canonical HTML: https://test.network/docs/wallet · This Markdown is served for `Accept: text/markdown`.
More for agents: [llms.txt](https://test.network/llms.txt) · [Full docs as Markdown](https://test.network/llms-full.txt) · [OpenAPI](https://test.network/openapi.json) · [Sitemap](https://test.network/sitemap.xml)
