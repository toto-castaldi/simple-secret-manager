# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A single bash script, `ssm`. It encrypts and decrypts files using an X25519 key pair derived deterministically from a seed the user types in. It relies only on standard tools: `bash`, `openssl` 3.x, `od`, `head`, `tail`, `stat`. Don't add other dependencies (not even `xxd`). User-facing text (messages, README) is in Italian.

`README.md` documents usage, the crypto scheme and the `.ssm` file format. Keep it in sync when any of these change.

## Testing

The project has no test suite, build or lint step. `ssm` reads the seed from `/dev/tty`, so piping into stdin doesn't work. To run it non-interactively, use `script` to provide a pseudo-terminal:

```bash
S=$PWD/ssm
# encrypt (seed + confirmation), then decrypt and compare
(sleep 1; printf 'seed\n'; sleep 0.5; printf 'seed\n'; sleep 3) | script -qec "$S encrypt t.bin" /dev/null
mv t.bin t.orig
(sleep 1; printf 'seed\n'; sleep 3) | script -qec "$S decrypt t.bin.ssm" /dev/null
cmp t.orig t.bin
```

Run these in the scratchpad, not in the repo. Also check that a wrong seed and a tampered file both fail with `seed errato o file corrotto` and produce no output file.

## Invariants that must hold

- **Secrets never appear in argv or on disk.** The seed, private key, encryption password and MAC key reach `openssl` only through pipes and file descriptors: `-pass fd:3`, process substitution `<(hex2bin ...)`, or stdin.
  - Never pass them as command-line arguments to external commands (e.g. `-K`, `-kdfopt pass:`, `-hmac`), because those show up in `ps`.
  - Never use here-strings or here-docs for them either, since bash may back those with temp files.
  - `hex2bin` uses only the `printf` builtin for this reason. Bash variables can't hold NUL bytes, so all binary values are kept as hex strings and converted at the pipe boundary.
- **Compatibility with existing files.** Changing `KDF_SALT`, `KDF_ITER`, `MAGIC_HEX`, the PBKDF2 parameters, the `"enc"`/`"mac"` context strings, or the order of `shared || eph_pub || pub` in `derive_session_keys` makes every previously encrypted file unreadable. A format change needs a new magic (e.g. `SSM2`) and should keep decryption support for the old one.
- **Authenticate before decrypting.** `cmd_decrypt` checks the SHA3-256 tag over the whole file except the tag before it writes any plaintext. Output goes to a `mktemp` file that is moved into place, and existing files are never overwritten.

## Crypto structure (spread across functions in `ssm`)

1. `derive_private_key`: seed → PBKDF2 with a fixed salt, using `openssl enc -P`. The `key=` value becomes the 32-byte X25519 private key.
2. Raw 32-byte keys become DER by prepending the constant `PKCS8_PREFIX` / `SPKI_PREFIX`. `public_key` and `x25519` (`openssl pkeyutl -derive`) work on those DER blobs.
3. `cmd_encrypt` generates an ephemeral key with `openssl genpkey`. Both sides run `derive_session_keys` to get `ENC_PASS` (passed to `openssl enc -aes-256-ctr -pbkdf2 -iter 1`) and `MAC_KEY`. The MAC is `SHA3-256(mac_key || data)`, which is safe because SHA3 is not vulnerable to length extension.
4. File layout: `SSM1` (4 bytes) || eph_pub (32) || openssl enc output || tag (32). The offsets are `HEADER_LEN` and `TAG_LEN`.
