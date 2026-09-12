**Systems never store your real password — they store a _hash_:** a scrambled, one-way fingerprint of it. "Cracking" means guessing passwords, hashing each guess, and checking if it matches the stolen hash. You're not reversing the hash; you're racing through millions of guesses.

**Two situations, two approaches:** if you have a _stolen hash_ → crack it offline with `hashcat`/`john` (fast, silent, no lockouts). If you have a _login form / service_ and no hash → brute-force online with `hydra` (slower, noisy, can lock accounts — use sparingly).

**The first step is always identifying the hash type** so you pick the right hashcat mode. `rockyou.txt` is your go-to wordlist; for OSCP you rarely need anything fancier.