# simple-secret-manager

`ssm` è uno script bash che cifra e decifra file con una coppia di chiavi
asimmetrica (X25519) derivata in modo deterministico da un seed. La chiave
non viene mai scritta su disco: esiste solo in memoria per la durata del
comando.

Usa solo strumenti standard: `bash`, `openssl` (3.x), `od`, `head`, `tail`, `stat`.

## Uso

```bash
./ssm encrypt segreti.txt              # chiede il seed due volte, crea segreti.txt.ssm
./ssm decrypt segreti.txt.ssm          # chiede il seed, ricrea segreti.txt
./ssm decrypt segreti.txt.ssm -        # stampa il contenuto su stdout
./ssm encrypt segreti.txt out.ssm      # output esplicito
```

- Il seed viene letto da `/dev/tty` senza eco.
- Il file originale non viene cancellato dopo la cifratura (usa `shred -u` se serve).
- Lo script non sovrascrive mai file esistenti.
- Con seed errato o file manomesso esce con errore senza produrre output.

## Come funziona

```
priv     = PBKDF2-SHA256(seed, salt fisso "ssm-kdf1", 600000 iter)  -> chiave X25519
pub      = chiave pubblica di priv
eph      = coppia X25519 effimera, nuova a ogni cifratura
shared   = X25519(eph_priv, pub)            # in decifratura: X25519(priv, eph_pub)
enc_key  = SHA3-256("enc" || shared || eph_pub || pub)
mac_key  = SHA3-256("mac" || shared || eph_pub || pub)
```

Formato del file `.ssm`:

| Offset | Lunghezza | Contenuto                                   |
|--------|-----------|---------------------------------------------|
| 0      | 4         | magic `SSM1`                                |
| 4      | 32        | chiave pubblica effimera                    |
| 36     | variabile | output di `openssl enc -aes-256-ctr`        |
| fine-32| 32        | tag `SHA3-256(mac_key \|\| tutto il resto)` |

Il tag viene verificato prima di decifrare.

## Sicurezza

- **Nessun segreto su disco o in `ps`:** seed e chiavi stanno in variabili
  bash e arrivano a `openssl` solo via pipe e file descriptor (`-pass fd:3`,
  `<(...)`), mai come argomenti. I core dump sono disabilitati.
- **Swap:** la memoria del processo può finire in swap; usa una swap cifrata
  se è un problema.
- **Il seed è l'unica protezione.** Il salt è fisso (serve per avere una
  chiave deterministica), quindi un attaccante con il file cifrato può
  tentare il brute force del seed offline. Usa una passphrase lunga
  (es. 5–6 parole casuali).
- Cambiare `KDF_SALT`, `KDF_ITER` o lo schema di derivazione rende
  illeggibili i file cifrati in precedenza.
