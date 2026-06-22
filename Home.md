# VirtualEstate — Digitalni dvojček nepremičninskega trga

VirtualEstate je študentski projekt, razvit v sklopu predmetov **Spletno programiranje**, **Principi programskih jezikov** in **Sistemska administracija** na **FERI**. Projekt predstavlja digitalni dvojček slovenskega nepremičninskega trga, sistem, ki združuje **podatke iz javnih spletnih virov** (oglasniki nepremičnin) in **ročno vnesene podatke** v enotno bazo, jih vizualizira na zemljevidu in v grafih ter omogoča realnočasovno spremljanje sprememb.

## Komponente sistema

| Komponenta | Tehnologija | Vloga |
|---|---|---|
| **WebService** | Node.js, Express, MongoDB, Socket.IO | REST API + WebSocket posrednik |
| **WebClient** | React, Vite, Leaflet, Recharts | Spletni vmesnik za pregled in administracijo |
| **DesktopApplication** | Kotlin, Jetpack Compose | Namizna aplikacija za upravljanje baze, scraping in generiranje podatkov |

## Projektna dokumentacija

Glavna dokumentacija je razdeljena na štiri sklope:

1. [Projektne specifikacije](Projektne-specifikacije) — namen, uporabniki, opis rešitve in funkcionalne zahteve
2. [Navodila za namestitev in prijavo](Namestitev-in-prijava) — kako pognati sistem in se prijaviti
3. [Ključni primeri uporabe](Primeri-uporabe) — pet scenarijev z navodili po korakih
4. [Dokumentacija izvedenih lastnosti](Izvedene-lastnosti) — opis ključnih implementiranih funkcionalnosti

Pregled vseh strani: [Projektno dokumentacijo](Documentation).

## Repozitorij in povezave

- **Glavni repozitorij**: https://github.com/andrej-nemanic/VirtualEstate_Project
- **Wiki vir**: https://github.com/NikSignjarZilavec/VirtualEstate-Wiki (samodejna sinhronizacija prek CircleCI)
- **Issue tracker**: Jira (SCRUM)
- **Containerji**: DockerHub
- **Deployment**: Microsoft Azure

## Avtorji

- **Nik Signjar Zilavec** — desktop aplikacija, spletni vmesnik, scraping, generator, UI/UX
- **Andrej Nemanič** — spletna storitev, baza, deployment, dokumentacija
