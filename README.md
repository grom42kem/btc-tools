# Bitcoin Address Tools

A web tool for converting between Bitcoin formats (hash160, addresses, private/public keys) and deriving a TRON address from a keypair. Runs entirely in the browser; no data is sent to any server.

**Live (GitHub Pages):** [https://grom42kem.github.io/btc-tools/](https://grom42kem.github.io/btc-tools/)

---

## Tabs and logic

### 1. Hash160 → Addresses

**Input:** 20-byte hash160 in hex (40 characters 0-9, a-f).

**What it does:**
- The same hash160 is encoded into three formats for **mainnet** and **testnet**:
  - **P2PKH** (Pay to Public Key Hash) — Legacy, prefix `1` (mainnet) or `m/n` (testnet). Base58Check encoding: version 0x00 (mainnet) or 0x6f (testnet) + 20-byte hash160.
  - **P2SH** (Pay to Script Hash) — prefix `3` (mainnet) or `2` (testnet). Base58Check, version 0x05 (mainnet) or 0xc4 (testnet) + same 20-byte hash160.
  - **Bech32** (SegWit P2WPKH) — prefix `bc1q` (mainnet) or `tb1q` (testnet). Bech32 encoding (BIP 173): HRP `bc`/`tb`, witness version 0, 20-byte witness program.
- TRON address (TRX): the same 20 bytes are used as the account id, prefixed with `0x41`, then Base58Check-encoded and linked to Tronscan.

**Output:** Six Bitcoin addresses (3 mainnet, 3 testnet) plus a TRON (TRX) address, all with explorer links and Copy buttons.

---

### 2. Address → Hash160

**Input:** A single Bitcoin or TRON address.

**What it does:**
- **Bech32** (`bc1q...`, `tb1q...`): Decode Bech32 → HRP + witness version + data; verify checksum (polymod). Only witness version 0 and 20-byte program (P2WPKH) are supported. Hash160 = those 20 bytes.
- **Bitcoin Base58Check** (`1...`, `3...`, `m...`, `2...`): Decode Base58 → payload; verify checksum (double SHA-256, last 4 bytes). First byte is version (0x00, 0x6f, 0x05, 0xc4); remaining 20 bytes are hash160.
- **TRON Base58Check** (`T...`): Decode Base58 → payload; verify checksum; version byte `0x41` + 20-byte account id. Those 20 bytes are returned as hex (TRON account id).

**Output:** One 20-byte hex string (hash160 / account id) with Copy button and an explorer link (Blockchain.com for Bitcoin, Tronscan for TRON).

---

### 3. Private key → Address

**Input:** Private key as:
- **Hex:** 64 characters (32 bytes);
- **WIF:** Base58Check (starts with `5`, `L`, `K` for mainnet or `9`, `c` for testnet); if payload is 33 bytes, last byte 0x01 indicates compressed key.

**What it does:**
1. Parse: from hex → 32 bytes; from WIF → Base58Check decode, check version (0x80 mainnet / 0xef testnet), take 32-byte key from payload. The WIF compression flag is respected (uncompressed WIF derives an uncompressed Bitcoin pubkey).
2. Public key (Bitcoin): compressed or uncompressed on secp256k1 via [@noble/secp256k1](https://github.com/paulmillr/noble-secp256k1).
3. Hash160: `RIPEMD160(SHA256(public_key))` — SHA-256 via `crypto.subtle`, RIPEMD160 via [@noble/hashes](https://github.com/paulmillr/noble-hashes).
4. Same as tab “Hash160 → Addresses”: all six addresses (Mainnet/Testnet × P2PKH, P2SH, Bech32) with links and Copy.
5. TRON address: from the **uncompressed** public key using `keccak256(x||y)` (64-byte body, without the `04` prefix), take the last 20 bytes, prefix with `0x41`, then Base58Check-encode. Output is linked to Tronscan.

**Output:** Same Bitcoin address blocks as in section 1, plus a **TRON** section (TRX address) with Tronscan link and Copy.

---

### 4. Public key → Addresses

**Input:** Public key in hex:
- **Compressed:** 66 characters — prefix `02` or `03` + 32 bytes (x coordinate).
- **Uncompressed:** 130 characters — prefix `04` + 32 bytes x + 32 bytes y.

**What it does:**
1. Validate length and prefix (02/03/04).
2. Hash160: `RIPEMD160(SHA256(public_key))` — same libraries (RIPEMD160 from @noble/hashes, SHA-256 from `crypto.subtle`).
3. Same as section 1: generate six addresses (Mainnet/Testnet × P2PKH, P2SH, Bech32).
4. TRON address: ensure an uncompressed public key (decompress if needed), then derive TRX address as described above and link to Tronscan.

**Output:** Same Bitcoin address blocks with explorer links and Copy buttons, plus a **TRON** section (TRX address) with Tronscan link and Copy.

---

## Technical details

| Component | Implementation |
|-----------|----------------|
| **Base58 / Base58Check** | Inline code: Base58 alphabet, BigInt for encode/decode; checksum = last 4 bytes of SHA-256(SHA-256(payload)). |
| **Bech32** | Inline code per BIP 173: polymod, HRP expand, 8-bit ↔ 5-bit word conversion, checksum. |
| **SHA-256** | `crypto.subtle.digest('SHA-256', data)` (browser API). |
| **RIPEMD-160** | @noble/hashes (dynamic import from esm.sh). |
| **Keccak-256** | @noble/hashes `sha3` (dynamic import from esm.sh) — used for TRON address derivation. |
| **secp256k1** | @noble/secp256k1 (dynamic import from esm.sh) — used to derive public keys from private keys and to decompress compressed public keys for TRON. |

All computation runs locally in the browser; keys and addresses are never sent anywhere.

---

## Copy and links

- Every generated value (address, hash160) has a **Copy** button — copies to clipboard and briefly shows “Copied!”.
- Addresses link to [Blockchain.com Explorer](https://www.blockchain.com/explorer/addresses/btc/) as:  
  `https://www.blockchain.com/explorer/addresses/btc/{address}`
- TRON addresses link to Tronscan as:  
  `https://tronscan.org/#/address/{address}`

---

## Run locally

Open `index.html` in a browser or serve the repo root with any static server (dynamic imports from esm.sh require internet access).

## GitHub Pages

The repo is set up to publish from the `main` branch, root directory. After push, the site is available at the link above.
