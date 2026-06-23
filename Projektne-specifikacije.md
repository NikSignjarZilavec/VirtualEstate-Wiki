# 1. Projektne specifikacije

## 1.1 Namen projekta

**VirtualEstate** je digitalni dvojček nepremičninskega trga. Njegov osnovni namen je centralizirati in poenotiti razpršene podatke o nepremičninskih oglasih, jih vizualizirati v formatu, ki uporabnikom omogoča hiter pregled in primerjavo, ter zagotoviti administrativno orodje za ročno upravljanje baze.

Projekt naslavlja dva temeljna problema:

1. **Razdrobljenost trga.** Nepremičnine v Sloveniji oglašuje več neodvisnih portalov (`nepremicnina.si`, `24nep.si`, …). Iskanje po več portalih hkrati je zamudno, primerjava cen in lastnosti pa otežena.
2. **Pomanjkanje vizualizacij.** Večina oglasnih portalov ponuja zgolj seznamski prikaz, brez naprednih grafov ali zemljevida z območnimi filtri, ki bi uporabniku omogočili razumevanje trga kot celote.

### 1.1.1 Skupine uporabnikov in njihove potrebe

| Skupina | Tipičen profil | Glavne potrebe |
|---|---|---|
| **Iskalec nepremičnine** | Posameznik ali družina, ki išče stanovanje/hišo za nakup ali najem | Hitro filtriranje po regiji, ceni, velikosti; iskanje po opisu ("balkon", "parking"); pregled lokacije na zemljevidu |
| **Investitor** | Posameznik ali podjetje, ki analizira tržne trende | Grafi povprečnih cen po tipu in regiji; pregled cen na večjem geografskem območju |
| **Administrator podatkov** | Skrbnik baze (npr. nepremičninska agencija) | Ročno dodajanje, urejanje in brisanje zapisov; razčlenjevanje podatkov iz spletnih virov v bazo; generiranje testnih podatkov |
| **Razvijalec / raziskovalec** | Programer, ki želi nadgraditi sistem ali analizirati podatke | REST API z geoprostorskimi poizvedbami; WebSocket dostop do realnočasovnih sprememb |

## 1.2 Opis rešitve

VirtualEstate sestavljajo **tri komponente**, ki komunicirajo prek REST API in WebSocket protokola:

```
┌─────────────────┐       ┌─────────────────┐       ┌───────────────────┐
│  WebClient      │       │  WebService     │       │  DesktopApp       │
│  (React, SPA)   │◄─────►│  (Node.js+      │◄─────►│  (Kotlin Compose) │
│                 │  REST │   Express+      │  REST │                   │
│                 │   +   │   MongoDB+      │       │                   │
│                 │  WS   │   Socket.IO)    │       │                   │
└─────────────────┘       └─────────────────┘       └───────────────────┘
```

### 1.2.1 Kako rešitev naslavlja izpostavljene potrebe

| Potreba | Pristop rešitve |
|---|---|
| Centralizacija razpršenih podatkov | Namizna aplikacija razčlenjuje (web scraping) podatke iz dveh slovenskih oglasnikov in jih prek REST endpointa (`POST /api/properties/ingest`) shrani v skupno MongoDB bazo. |
| Hitro filtriranje | Spletni vmesnik ponuja URL-sinhrone filtre (vrsta, regija, mesto, cena, velikost, opis) z odzivnim seznamom. |
| Iskanje po besednih nizih v opisu | Backend uporablja MongoDB `$regex` z escape-om meta-znakov za varno iskanje (npr. "balkon"). |
| Vizualizacija na zemljevidu | Leaflet zemljevid s clustering markerji in možnostjo risanja območja (`poligon`, `krog`); poizvedba uporabi `$geoWithin` in `$centerSphere` MongoDB operatorja. |
| Analitične vizualizacije | Recharts grafi: število nepremičnin po tipu (tortni), povprečna cena po tipu (stolpčni), razmerje cena-velikost (raztreseni). |
| Realnočasovne posodobitve | Socket.IO broadcasti dogodke `propertyCreated`, `propertyUpdated`, `propertyDeleted` vsem aktivnim odjemalcem. |
| Ročno upravljanje baze | Spletni admin vmesnik in namizna aplikacija s polnim CRUD-om za nepremičnine in uporabnike. |
| Generiranje testnih podatkov | Namizna aplikacija s parametri (število, razpon cene, razpon velikosti) generira namišljene nepremičnine na realnih slovenskih lokacijah. |

## 1.3 Funkcionalne zahteve

### 1.3.1 Spletna storitev (WebService)

| ID | Funkcionalnost | Tip |
|---|---|---|
| FZ-1.1 | CRUD operacije nad nepremičninami (`/api/properties`) | OBVEZNO |
| FZ-1.2 | CRUD operacije nad uporabniki (`/api/users`) | OBVEZNO |
| FZ-1.3 | JWT-žetonska avtentikacija s 24-urno veljavnostjo | OBVEZNO |
| FZ-1.4 | Vloga *admin* z ekskluzivnim dostopom do administrativnih endpointov | OBVEZNO |
| FZ-1.5 | WebSocket bridge za realnočasovne dogodke (Socket.IO) | OBVEZNO |
| FZ-1.6 | Geoprostorske poizvedbe: po radiju (`$near`), po območju (`$geoWithin`) | OBVEZNO |
| FZ-1.7 | Iskanje po opisu z regex (varno escape-anje meta-znakov) | OBVEZNO |
| FZ-1.8 | Paginacija (`page`, `limit`), urejanje (`sortBy`, `sortDir`) | RAZŠIRJENO |
| FZ-1.9 | Agregatne statistike (`/api/properties/stats`) za grafe | RAZŠIRJENO |
| FZ-1.10 | Samodejno geokodiranje naslovov prek Nominatim API-ja | RAZŠIRJENO |
| FZ-1.11 | Rate-limiting na prijavni endpoint (10 zahtevkov / 15 min) | RAZŠIRJENO |
| FZ-1.12 | Validacija MongoDB `ObjectId` v URL-jih | RAZŠIRJENO |

### 1.3.2 Spletni vmesnik (WebClient)

| ID | Funkcionalnost | Tip |
|---|---|---|
| FZ-2.1 | Pregled nepremičnin s paginacijo (2×3 = 6 zapisov/stran) | OBVEZNO |
| FZ-2.2 | Filtriranje po vrsti, ponudbi, regiji, mestu, ceni, velikosti, opisu | OBVEZNO |
| FZ-2.3 | URL-sinhroni filtri (deljive povezave) | OBVEZNO |
| FZ-2.4 | Interaktivni zemljevid (Leaflet) z markerji in clustering | OBVEZNO |
| FZ-2.5 | Risanje območja (poligon, krog) za geo-filter | OBVEZNO |
| FZ-2.6 | Grafi: pita po tipu, stolpčni po cenah, raztreseni cena-velikost | OBVEZNO |
| FZ-2.7 | Administrativni vmesnik z CRUD-om in iskanjem | OBVEZNO |
| FZ-2.8 | Realnočasovne posodobitve prek WebSocket-a | OBVEZNO |
| FZ-2.9 | Prijava, registracija in odjava | OBVEZNO |
| FZ-2.10 | Preklop med svetlim in temnim načinom | RAZŠIRJENO |
| FZ-2.11 | Polno odzivni vmesnik (mobile-first) | RAZŠIRJENO |
| FZ-2.12 | Sortiranje stolpcev v administracijski tabeli | RAZŠIRJENO |

### 1.3.3 Namizna aplikacija (DesktopApplication)

| ID | Funkcionalnost | Tip |
|---|---|---|
| FZ-3.1 | Vnašanje, posodabljanje, brisanje in pregled vseh tabel | OBVEZNO |
| FZ-3.2 | Razčlenjevanje (scraping) iz dveh spletnih virov | OBVEZNO |
| FZ-3.3 | Filtriranje razčlenjenih podatkov pred pošiljanjem v bazo | OBVEZNO |
| FZ-3.4 | Generiranje namišljenih podatkov s parametri (število, razponi) | OBVEZNO |
| FZ-3.5 | Komunikacija z bazo izključno prek REST API-ja | OBVEZNO |
| FZ-3.6 | Sortiranje, paginacija in iskanje v vseh tabelah | RAZŠIRJENO |
| FZ-3.7 | Skupinsko (bulk) brisanje izbranih zapisov | RAZŠIRJENO |
| FZ-3.8 | Validacija polj v urejevalnih oknih | RAZŠIRJENO |

## 1.4 Nefunkcionalne (sistemske) zahteve

| Kategorija | Zahteva |
|---|---|
| **Varnost** | Gesla so shranjena z `bcrypt` (cost 10). JWT podpisan z `HS256`. Prijavni endpoint omejen z rate-limit-om. Vsi regex inputi so escape-ani. |
| **Performanca** | MongoDB indeksi nad poljami `email` (unique) in `coordinates` (2dsphere). Geo-poizvedbe na realnih datasetih ≤ 100 ms. Spletni vmesnik nalaganje pod 2 s na 4G. |
| **Skalabilnost** | Stateless API; horizontalno skaliranje prek Docker/Kubernetes. Socket.IO. |
| **Razširljivost** | Mongoose modeli omogočajo enostavno dodajanje atributov (npr. `priceHistory`, `images`). REST endpointi sledijo standardnemu vzorcu. |
| **Združljivost** | Spletni vmesnik testiran na Chrome, Firefox, Edge in Safari. Namizna aplikacija deluje na Windows, macOS in Linux (JVM 17+). |
| **Razpoložljivost** | Storitev je razmeščena na Microsoft Azure z avtomatičnim CI/CD prek GitHub Actions in DockerHub. |
| **Vzdrževanje** | Strukturirana koda po MVC vzorcu (WebService) in feature-based arhitekturi (WebClient, Desktop). Logging z `pino` na backendu. |
| **Dostopnost** | ARIA atributi (`aria-label`, `aria-live`, `aria-sort`) na ključnih komponentah; touch-targets ≥ 40 px na mobilnih napravah. |

## 1.5 Tehnološki sklad

### Backend
- **Node.js** 20 LTS
- **Express** 4.x — HTTP server in routing
- **Mongoose** 9.x — ODM za MongoDB
- **MongoDB** 6+ — dokumentna baza
- **Socket.IO** 4.x — WebSocket komunikacija
- **bcrypt**, **jsonwebtoken** — varnost
- **pino** — logging
- **express-rate-limit** — DoS zaščita

### Frontend
- **React** 18 + **Vite** 5
- **React Router** 6 — navigacija
- **Axios** — HTTP odjemalec
- **Leaflet** + **react-leaflet** + **leaflet-draw** + **react-leaflet-cluster** — zemljevid
- **Recharts** — grafi
- **Socket.IO client** — WebSocket

### Desktop
- **Kotlin** 1.9 + **Jetpack Compose** Multiplatform
- **Retrofit** + **OkHttp** — REST komunikacija
- **Gson** — JSON serializacija
- **Ksoup** — HTML parsing za scraping

### Infrastruktura
- **Docker** + **Docker Compose** — kontejnerizacija
- **GitHub Actions** — CI
- **CircleCI** — wiki sinhronizacija
- **DockerHub** — registry
- **Microsoft Azure** — deployment
- **MongoDB Atlas** — managed baza (opcijsko)

## 1.6 Slovar pojmov in kratic

| Pojem / kratica | Pomen |
|---|---|
| **SPA** | Single-Page Application — spletni vmesnik, ki teče v eni naloženi strani (React + Vite). |
| **REST API** | Representational State Transfer — HTTP vmesnik za CRUD operacije nad viri. |
| **WebSocket / Socket.IO** | Persistentna dvosmerna povezava za realnočasovne dogodke. |
| **JWT** | JSON Web Token — podpisan žeton za stateless avtentikacijo. |
| **CRUD** | Create, Read, Update, Delete — osnovne operacije nad zapisi. |
| **ODM** | Object-Document Mapper (Mongoose) — preslikava med JS objekti in Mongo dokumenti. |
| **2dsphere** | MongoDB geoprostorski indeks za poizvedbe nad koordinatami na sferi. |
| **GeoJSON Point** | Format `{ type: 'Point', coordinates: [lng, lat] }` za shranjevanje lokacije. |
| **Geokodiranje** | Pretvorba naslova (regija/mesto/naselje) v koordinate prek Nominatim API-ja. |
| **Scraping** | Samodejno razčlenjevanje podatkov iz HTML strani spletnih virov. |
| **Nominatim** | Brezplačni OpenStreetMap geokodirni servis. |
| **Bounding box (bbox)** | Pravokotno geografsko območje, podano s štirimi koordinatami. |
| **Ingest** | Vnos zapisov v bazo prek `/api/properties/ingest` endpointa (scraping/generator). |
| **FZ-x.x** | Oznaka funkcionalne zahteve v tem dokumentu (poglavje 1.3). |
| **SCRUM-XX** | Oznaka Jira podopravila (issue), prek katere sledimo implementaciji. |

## 1.7 Sledljivost zahtev (traceability)

Spodnja tabela povezuje **funkcionalne zahteve** (poglavje 1.3) s **primeri uporabe** ([Primeri uporabe](Primeri-uporabe)) in **izvedenimi lastnostmi** ([Izvedene lastnosti](Izvedene-lastnosti)). Tako je mogoče preveriti pokritost vsake zahteve od specifikacije do implementacije.

| Funkcionalne zahteve | Primer uporabe | Izvedena lastnost |
|---|---|---|
| FZ-1.3, FZ-1.4, FZ-2.9 — avtentikacija, admin vloga, prijava/registracija | Primer 3 | Lastnost 1 |
| FZ-1.5, FZ-2.8 — realnočasovne posodobitve (WebSocket) | Primer 3 | Lastnost 2 |
| FZ-1.6, FZ-2.4, FZ-2.5 — geoprostorske poizvedbe, zemljevid, območni filter | Primer 2 | Lastnost 3 |
| FZ-1.7, FZ-2.2 — iskanje po opisu z varnim regex | Primer 1 | Lastnost 4 |
| FZ-3.2, FZ-3.3, FZ-3.5 — razčlenjevanje s spleta, filtriranje, REST komunikacija | Primer 4 | Lastnost 5 |
| FZ-3.4 — generiranje testnih podatkov | Primer 5 | Lastnost 6 |
| FZ-1.9, FZ-2.6 — agregatne statistike in grafi | Primer 1 | Lastnost 7 |
| FZ-2.10, FZ-2.11 — temni/svetli način, odzivni vmesnik | vsi primeri (UI) | Lastnost 8 |
