# OverTheWire — Krypton Writeup

My walkthrough of all 8 levels (0 → 7) of [OverTheWire: Krypton](https://overthewire.org/wargames/krypton), the wargame that teaches classical cryptography — from spotting that base64 isn't encryption, through Caesar and substitution ciphers, up to breaking Vigenère and a repeating-key stream cipher.

**A note on passwords:** Following OverTheWire's own guidance, I don't publish level passwords — it spoils the challenge for others. This writeup documents the cipher, the weakness, and the attack for each level. Run the techniques yourself to recover each password. `<kryptonN-password>` stands in for the credential you carry from the previous level.

## How to connect

Levels 1–7 are reached over SSH:

```bash
ssh kryptonN@krypton.labs.overthewire.org -p 2231
```

Each level's material lives under `/krypton/kryptonN/` (README, intercepted ciphertexts, and the encrypted password for the next level). The starting credential for `krypton0` is base64-encoded on the Krypton homepage.

A working directory in `/tmp`, plus `tr`, and a Vigenère/substitution solver (dcode.fr, quipqiup, or a short Python script) is all you need.

## Levels

### Level 0 → 1 — Base64 (encoding ≠ encryption)

The homepage hands you a base64 blob. It looks scrambled but carries no key — it's just an encoding. Decode it to get `krypton1`'s SSH password:

```bash
echo "<base64-string-from-site>" | base64 -d
```

Lesson: base64 is reversible by anyone; it protects nothing.

### Level 1 → 2 — ROT13

In `/krypton/krypton1/`, the file `krypton2` holds the next password encoded with ROT13 (word boundaries preserved). ROT13 is its own inverse — rotate every letter 13 places:

```bash
cat /krypton/krypton1/krypton2 | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

### Level 2 → 3 — Caesar cipher via an encryption oracle

`/krypton/krypton2/` contains the ciphertext `krypton3` (5-letter groups), a setuid `encrypt` binary, and `keyfile.dat`. It's a Caesar shift, but the shift is unknown. The trick: the `encrypt` binary uses the same key and reads `keyfile.dat` from the current directory — so it's a free known-plaintext oracle.

```bash
cd $(mktemp -d)          # work dir in /tmp
chmod 777 .              # encrypt runs setuid krypton3, so it needs access
ln -s /krypton/krypton2/keyfile.dat keyfile.dat
echo "AAAAAAAAAA" > known.txt
/krypton/krypton2/encrypt known.txt   # produces ./ciphertext
cat ciphertext           # where 'A' lands tells you the shift
```

Read the shift off the encrypted `A`s, then reverse that shift on `krypton3` with `tr` (or dcode's Caesar tool).

### Level 3 → 4 — Monoalphabetic substitution (frequency analysis)

`/krypton/krypton3/` gives you three intercepted ciphertexts (`found1`, `found2`, `found3`) in the same cipher, plus `krypton4`. There's no key and no oracle, but a monoalphabetic substitution preserves letter frequencies. Concatenate the `found` files for a large sample, run frequency analysis (English runs E, T, A, O, I, N…), build the letter mapping, and apply it to `krypton4`. A solver like quipqiup or dcode's substitution breaker gets you there fast.

### Level 4 → 5 — Vigenère, known key length

`/krypton/krypton4/` provides `found1` and a README that tells you the key length is 6. Vigenère is just interleaved Caesar ciphers, so split the ciphertext into 6 columns — every 6th letter shares one key letter — and solve each column by frequency analysis. That recovers the keyword; apply it to decrypt `krypton5`. (dcode or the Black Chamber Vigenère tool, set to key length 6, automates the column analysis.)

### Level 5 → 6 — Vigenère, unknown key length

`/krypton/krypton5/` gives more intercepts and `krypton6`, but this time the key length is not given — you have to find it first:

- **Kasiski examination:** look for repeated ciphertext substrings; the gaps between them are multiples of the key length.
- **Index of Coincidence / Friedman test:** estimate the key length statistically.

Once you know the length, it collapses to the level-4 problem: per-column frequency analysis → keyword → decrypt `krypton6`. An automated Vigenère solver will determine both the length and the key.

### Level 6 → 7 — Periodic stream cipher (keystream recovery)

The hardest one. `/krypton/krypton6/` contains a README, the `encrypt6` binary, `keyfile.dat`, a sample plaintext/ciphertext pair, and the ciphertext `krypton7`. The cipher applies a repeating keystream — a longer periodic polyalphabetic key. Repeating-key ciphers fall completely to known plaintext: encrypting a run of identical characters exposes the keystream directly, because a constant input makes the output trace the key itself.

```bash
cd $(mktemp -d); chmod 777 .
ln -s /krypton/krypton6/keyfile.dat keyfile.dat
python3 -c "print('A'*120)" > known.txt
/krypton/krypton6/encrypt6 known.txt   # output reveals the repeating key
```

Encrypt a long block of `A`s, read the repeating pattern in the output to find the period and recover the key sequence, then invert that keystream over `krypton7`'s ciphertext to get the final password.

### Level 7 — The end

`Congratulations, you have reached the end.` 🎉

## Cryptography concepts covered

- **Encoding vs. encryption** — base64 carries no key and protects nothing
- **Caesar / ROT13** — fixed-shift substitution and its trivial reversal
- **Encryption oracles & known-plaintext attacks** — using an encrypt binary to leak the key
- **Monoalphabetic substitution & frequency analysis** — letter-frequency fingerprints
- **Vigenère (polyalphabetic)** — column analysis with a known key length
- **Key-length recovery** — Kasiski examination and the Index of Coincidence / Friedman test
- **Repeating-key stream ciphers** — why key reuse plus predictable plaintext is fatal

The through-line: every cipher here leaks because of structure the attacker can predict — stable letter frequencies, a reusable key, or an oracle that encrypts chosen input.

---

Part of my [OverTheWire](https://overthewire.org/wargames/) series. Bandit and Natas done; Leviathan and Narnia to follow.
