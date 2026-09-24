# SecureVault — Secure File Distribution & Reconstruction System

**Split any file into encrypted, self-healing fragments. Lose some. Rebuild the original from whatever survives — byte for byte.**

A complete reimagining of the C++ file splitter/merger: a cross-platform Python crypto engine, a FastAPI control plane, and an interactive **3D WebGL interface** where your file is a crystal core that *shatters into numbered shards* and *reassembles* when you reconstruct it.

| | |
|---|---|
| Frontend | Vanilla JS + Three.js (WebGL, vendored — works fully offline) |
| Backend | Python · FastAPI · uvicorn |
| Crypto | AES-256-CBC · scrypt KDF · Shamir's Secret Sharing (GF(2⁸)) · Reed-Solomon erasure coding · SHA-256 everywhere |
| Cloud | Any **S3-compatible** store (AWS S3, Cloudflare R2, Backblaze B2, MinIO, Wasabi) and any **WebDAV** server (Nextcloud, ownCloud, Synology, Box) — pure stdlib, zero SDK dependencies |

---

## What it does

1. **Shatter** — pick any file (any type, any size), choose `n` shards and a threshold `k`. The file is:
   * erasure-coded with a systematic **Cauchy Reed-Solomon matrix**, so **any k of the n shards** rebuild the file (the other n−k act as parity you may lose), and
   * optionally **password-protected**: the content key is derived with **scrypt** (or, in threshold mode, **Shamir-split** across the shards — fewer than k shards reveal *neither data nor key*).
2. **Store** — shards go to the managed vault, **any folder you choose**, and/or get pushed to your **cloud storage** under `securevault/<set-id>/`.
3. **Reconstruct** — drop *any* k shards (from disk or pulled straight from the cloud), the engine rebuilds the file and **verifies SHA-256 integrity end-to-end**.

Every fragment is **self-describing** (a JSON header rides inside it) — no manifest file needed to reconstruct.

## The 3D UI

![Hero — the vault core](docs/screenshots/01-hero-3d-core.png)

Splitting triggers the shatter sequence; reconstructing pulls the shards home:

| Shatter | Reconstruct |
|---|---|
| ![Shatter](docs/screenshots/02-shatter-animation.png) | ![Reconstruct](docs/screenshots/03-reconstruct-animation.png) |

Numbered hex tags float on each shard (hover / click them), key-share shards glow with a cyan halo, and password-protected sets switch the palette to gold. Orbit with drag, zoom with scroll.

## Quick start

```bash
pip install fastapi uvicorn python-multipart cryptography   # that's everything
cd server
python -m uvicorn main:app --port 8000
# open http://127.0.0.1:8000
```

No build step. No database. No JS bundler. Three.js is vendored in `frontend/vendor/`.

## Cloud distribution

![Cloud connections](docs/screenshots/04-cloud-connections.png)

Add a connection in the **Cloud** tab (test & save), then push any set from the Vault panel or straight after a split. In the Reconstruct panel, *gather shards from cloud* lists your remote shards — select any k and reconstruct directly.

* **S3-compatible**: endpoint + bucket + region + access/secret keys. Requests are signed with **AWS Signature V4** (validated against the official AWS test vector).
* **WebDAV**: server URL + username + password/app-token (PROPFIND/PUT/GET/DELETE with basic auth).

Credentials are stored server-side in `cloud_connections.json` (git-ignored) and never returned to the browser unmasked.

## Vault & audit

![Vault and audit trail](docs/screenshots/05-vault-audit.png)

The Vault lists every fragment set (with per-shard download chips, cloud badges, custom-location awareness) and every reconstructed file, plus a live **audit trail** of all operations.

## How it works

```
┌──────────── frontend (Three.js scene + glass UI) ────────────┐
│  shatter  ·  reconstruct (disk or cloud)  ·  vault  ·  cloud │
└──────────────────────────────┬────────────────────────────────┘
                               │ HTTP (JSON / multipart)
┌──────────────────────────────▼────────────────────────────────┐
│  FastAPI  —  /api/split /api/merge /api/inspect /api/vault     │
│              /api/cloud/* /api/download/* /api/audit /api/health│
└──────┬───────────────────────────────────────────┬────────────┘
       │                                           │
┌──────▼──────── engine.py (pure Python) ──┐  ┌────▼──── cloud.py (stdlib) ────┐
│ RS erasure code (any-k-of-n data path)   │  │ S3 SigV4 · WebDAV basic-auth   │
│ Shamir GF(2⁸) key sharing                 │  │ upload / list / fetch / delete │
│ AES-256-CBC + scrypt / EVP_BytesToKey     │  └────────────────────────────────┘
│ self-describing .sv fragments + sha-256   │
└───────────────────────────────────────────┘
```

### Fragment format

```
magic     b"SVFRG1\0"
hdr_len   uint32 LE
header    JSON  — original name, size, part i/n, threshold k, mode,
          IV, KDF params (or Shamir share), per-shard & whole-file SHA-256
payload   AES-256-CBC ciphertext (or plaintext in "none" mode)
```

### Reconstruction rules

* any **k** of **n** shards suffice — tested exhaustively (all C(5,3) = 10 subsets of a 3-of-5 set reconstruct byte-identically);
* fewer than k → rejected, and in Shamir mode the key is mathematically unrecoverable;
* wrong password → clean `400 Wrong password`, no partial output;
* every merged file is checked against the whole-file SHA-256 → `integrity: verified | MISMATCH`.

## API (v2)

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/split` | fragment a file (`parts`, `threshold`, `mode=auto\|none\|password\|shamir`, `password`, `destination` folder) |
| POST | `/api/inspect` | parse fragment headers (drop-zone preview) |
| POST | `/api/merge` | reconstruct from uploaded shards (+password) |
| GET/DELETE | `/api/vault`, `/api/vault/{id}` | list / destroy fragment sets & merged files |
| GET | `/api/download/fragment/{set}/{name}` · `/api/download/merged/{name}` | downloads |
| GET/POST/DELETE | `/api/cloud`, `/api/cloud/{name}` | manage connections (save + test) |
| POST | `/api/cloud/{name}/upload/{set}` | push a set's shards to the cloud |
| GET | `/api/cloud/{name}/list` | list remote shards |
| POST | `/api/cloud/{name}/merge` | fetch selected remote shards & reconstruct |
| GET | `/api/audit` · `/api/stats` · `/api/health` | history / stats / status |
| POST | `/split`, `/merge` | **legacy v1 endpoints kept working** |

## Compatibility

The original C++/OpenSSL engine (`backend/`) still builds and runs where a Mach-O binary is usable; the Python engine also **merges raw `.enc` parts produced by the original C++ splitter** (EVP_BytesToKey SHA-256, 1 iteration, no salt). The legacy `/split` `/merge` endpoints use the C++ binary when present and fall back to the Python engine otherwise.

## Project layout

```
File_Splitter_Merger_CPP/
├── backend/          original C++ engine (splitter, crypto, main)
├── frontend/         3D UI — index.html, app.js, scene.js, style.css, vendor/three
├── server/
│   ├── main.py       FastAPI app
│   ├── engine.py     fragmentation / crypto engine
│   └── cloud.py      S3 (SigV4) + WebDAV client
└── docs/screenshots/
```

## Security notes

* Passwords never leave the page except to derive keys on the server; shards are encrypted **before** any cloud upload — only ciphertext leaves the machine.
* Per-fragment random IVs, scrypt (N=16384, r=8, p=1) key derivation, constant-time hash comparison (HMAC `compare_digest`).
* This is a demonstration-grade system: fragment confidentiality uses AES-256-CBC (not AEAD); for production consider AES-GCM and authenticated headers.
