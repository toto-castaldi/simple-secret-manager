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

Algoritmi e costruzione sono solidi, quindi la sicurezza dipende quasi solo da
quanto è difficile indovinare il seed. Chi ha il file cifrato può provare seed
offline, in parallelo, senza limiti di tentativi; non esiste una scorciatoia
per evitare di provarli uno per uno.

### Cosa non è il punto debole

- **Primitive:** X25519 (~128 bit), AES-256-CTR e MAC SHA3-256 sono standard,
  senza attacchi pratici noti.
- **Integrità:** il MAC impedisce di modificare il file senza che la
  decifratura se ne accorga.
- **Chiave effimera nuova a ogni cifratura:** due file cifrati non rivelano
  nulla l'uno dell'altro, nemmeno se hanno lo stesso contenuto.
- **Computer quantistici:** la chiave pubblica del destinatario non è nel file
  (deriva anch'essa dal seed), quindi anche un attaccante quantistico deve
  comunque indovinare il seed.

### Il vero punto debole: il seed

Ogni tentativo richiede PBKDF2-SHA256 con 600.000 iterazioni. Una GPU di
fascia alta fa circa 10–20 mila tentativi al secondo (stima di ordine di
grandezza, dai benchmark hashcat per PBKDF2-SHA256).

| Seed                            | Combinazioni | 1 GPU       | 1000 GPU    |
|---------------------------------|--------------|-------------|-------------|
| Parola o password comune        | dizionario   | secondi     | istantaneo  |
| 8 caratteri casuali [a-z0-9]    | ~2^41        | ~5 anni     | ~2 giorni   |
| 4 parole diceware casuali       | ~2^52        | millenni    | ~8 anni     |
| 6 parole diceware casuali       | ~2^78        | impossibile | impossibile |
| 12 caratteri casuali stampabili | ~2^79        | impossibile | impossibile |

Usa un seed **casuale** di almeno ~80 bit (6 parole diceware o 12–13
caratteri casuali). Un seed scelto "a mente" (frasi note, citazioni, password
riadattate) è il primo che un attaccante prova.

### Limiti di questo schema

- **Salt fisso:** è il prezzo della chiave deterministica. Un attaccante può
  precalcolare una tabella seed → chiave una volta sola e usarla contro tutti
  i file `ssm` di chiunque. Per un seed casuale e lungo non cambia nulla; per
  seed deboli rende l'attacco più economico.
- **PBKDF2 gira molto bene su GPU:** scrypt o Argon2 renderebbero ogni
  tentativo molto più costoso, ma con `openssl kdf` la password passerebbe
  come argomento, visibile in `ps`. È un compromesso voluto.
- Cambiare `KDF_SALT`, `KDF_ITER` o lo schema di derivazione rende
  illeggibili i file cifrati in precedenza.

### Rischi pratici

Spesso sono più probabili di un attacco crittografico.

- **Nessun segreto su disco o in `ps`:** seed e chiavi stanno in variabili
  bash e arrivano a `openssl` solo via pipe e file descriptor (`-pass fd:3`,
  `<(...)`), mai come argomenti. I core dump sono disabilitati.
- **Il file in chiaro:** l'originale resta su disco dopo la cifratura, e anche
  dopo un `rm` può restare recuperabile dal filesystem. Per leggere un file
  cifrato preferisci `decrypt ... -` a un file decifrato su disco.
- **Swap:** la memoria del processo può finire in swap; usa una swap cifrata.
- **La macchina:** il seed digitato su un sistema compromesso (es. keylogger)
  è perso.
