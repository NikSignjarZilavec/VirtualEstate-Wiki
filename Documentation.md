# Projektna dokumentacija

Dokumentacija projekta **VirtualEstate** je strukturirana skladno s standardnimi smernicami za pripravo zaključne dokumentacije študentskega projekta. Obsega štiri ločene sklope, ki skupaj pokrivajo vse vidike rešitve — od specifikacij in arhitekture do operativnih navodil in pregleda izvedenih funkcionalnosti.

## Vsebina

### 1. [Projektne specifikacije](Projektne-specifikacije)
Opredelitev namena študentskega projekta, ciljnih skupin uporabnikov in njihovih potreb, opis predlagane rešitve in pregled funkcionalnih ter nefunkcionalnih (sistemskih) zahtev.

### 2. [Navodila za namestitev in prijavo v sistem](Namestitev-in-prijava)
Korak-za-korakom navodilo za zagon vseh treh komponent (spletna storitev, spletni vmesnik, namizna aplikacija), pripravo okolja in prvo prijavo uporabnika.

### 3. [Ključni primeri uporabe](Primeri-uporabe)
Pet najpogostejših scenarijev uporabe rešitve, predstavljenih v obliki kratkih navodil (angl. *tutorial*) — kaj uporabnik želi narediti, koraki izvedbe in pričakovani rezultat.

### 4. [Dokumentacija izvedenih lastnosti](Izvedene-lastnosti)
Podroben opis ključnih implementiranih funkcionalnosti (feature/issue) s pojasnjenim namenom, načinom implementacije in navodili za uporabo.

---

## Dodatne vsebine v repozitoriju

Poleg te wiki dokumentacije so v glavnem repozitoriju shranjena tudi naslednja gradiva:

- **`Documentation/PRINCIPI/`** — poročili za predmet Principi programskih jezikov (Projektna naloga 1 in 2).
- **`Documentation/SPLETNO/`** — načrt za predmet Spletno programiranje.
- **`Documentation/SISTEMSKA/`** — gradivo za predmet Sistemska administracija (Docker, Azure, CI/CD, predstavitev).
- **`README.md`** — kratek pregled in zagon prek `docker compose`.

## Konvencije

- Vsi izrazi so zapisani v slovenščini, razen tehničnih imen knjižnic, končnic in identifikatorjev.
- Kode v code blockih so testirani na Windows 11 in Ubuntu 22.04 z Dockerjem.
- Vsi `git` ukazi predpostavljajo, da uporabljaš HTTPS s personal access tokenom ali SSH ključem.
