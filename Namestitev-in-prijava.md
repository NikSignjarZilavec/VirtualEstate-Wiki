# 2. Navodila za namestitev in prijavo v sistem

Ta stran vsebuje vsa navodila, ki jih novi uporabnik potrebuje za **zagon sistema VirtualEstate** in **prvo prijavo**. Sistem je sestavljen iz treh komponent, ki lahko tečejo neodvisno; najlažji način zagona je prek `docker compose`, ki sproži vse hkrati. Spodaj sta opisana oba pristopa.

## 2.1 Sistemske zahteve

| Komponenta | Različica | Opomba |
|---|---|---|
| **Node.js** | ≥ 20 LTS | za WebService in WebClient (ali samostojen Docker) |
| **npm** | ≥ 10 | priložen Node.js |
| **MongoDB** | ≥ 6 | lokalno ali MongoDB Atlas; Docker compose ga vključi |
| **JDK** | ≥ 17 | za namizno aplikacijo (Compose Multiplatform JVM target) |
| **Gradle** | ≥ 8.5 | priložen prek `gradlew` |
| **Git** | poljubna | za clone repozitorija |
| **Docker** + **Docker Compose** | poljubna | priporočeno za hitri zagon |
| **Brskalnik** | Chrome, Firefox, Edge, Safari (zadnji 2 različici) | za WebClient |

> 💡 Če nimaš nameščenega vsega zgoraj naštetega, lahko sistem poženeš samo z **Docker** + **Git** prek `docker compose` (poglavje 2.3).

## 2.2 Pridobivanje izvorne kode

```bash
git clone https://github.com/andrej-nemanic/VirtualEstate_Project.git
cd VirtualEstate_Project
```

Struktura repozitorija:

```
VirtualEstate_Project/
├── WebService/           # Node.js REST + WebSocket
├── WebClient/            # React SPA
├── DesktopApplication/   # Kotlin Compose
├── Documentation/        # Poročila in predstavitev
├── docker-compose.yml    # Vse skupaj v Dockerju
└── README.md
```

## 2.3 Hitri zagon prek Docker Compose (priporočeno)

1. V korenu repozitorija odpri `docker-compose.yml` in preveri/spremeni okoljske spremenljivke:
   - `DATABASE_URL` — povezava do Mongo
   - `JWT_SECRET` — naključen niz znakov (npr. iz `openssl rand -hex 32`)
2. Poženi:

   ```bash
   docker compose up --build
   ```

3. Po nekaj minutah bo dostopno:
   - **WebClient**: http://localhost:5173
   - **WebService API**: http://localhost:3000
   - **MongoDB**: lokalni Mongo na portu 27017 (ali tvoj Atlas, če nastaviš `DATABASE_URL`)

4. Za zaustavitev:

   ```bash
   docker compose down
   ```

## 2.4 Ročna namestitev (brez Dockerja)

Vsaka komponenta ima svoj `README.md` s podrobnejšimi navodili. Spodaj je povzetek.

### 2.4.1 WebService (spletna storitev)

```bash
cd WebService
cp .env.example .env       # uredi DATABASE_URL in JWT_SECRET
npm install
npm start
```

Privzeti port: **3000**. Preveri delovanje:

```bash
curl http://localhost:3000/
# {"name":"VirtualEstate API","version":"1.0.0"}
```

### 2.4.2 WebClient (spletni vmesnik)

V **novem terminalu**:

```bash
cd WebClient
npm install
npm run dev
```

Privzeti port: **5173**. Vite proxy avtomatsko preusmeri `/api` in `/socket.io` zahtevke na `http://localhost:3000` (glej `vite.config.js`). Aplikacija je dostopna na: http://localhost:5173

Za produkcijski build:

```bash
npm run build
npm run preview
```

### 2.4.3 DesktopApplication (namizna aplikacija)

V **novem terminalu**:

```bash
cd DesktopApplication
./gradlew :composeApp:run
```

(Na Windowsu uporabi `gradlew.bat`.)

Aplikacija se odpre v ločenem oknu. Privzeto se povezuje na `http://localhost:3000/api/`. Če imaš WebService na drugem naslovu, nastavi okoljsko spremenljivko **pred** zagonom:

```bash
# Windows PowerShell
$env:VIRTUALESTATE_API_URL = "http://moj-strežnik:3000/api/"
./gradlew :composeApp:run
```

```bash
# Linux/macOS
export VIRTUALESTATE_API_URL=http://moj-strežnik:3000/api/
./gradlew :composeApp:run
```

## 2.5 Prva prijava

Sistem nima predhodno ustvarjenega *seed* admin uporabnika — moraš ga ustvariti sam.

### 2.5.1 Ustvarjanje računa

1. Odpri http://localhost:5173/register
2. Vnesi:
   - **Ime** (poljubno)
   - **E-pošto** (mora biti edinstvena)
   - **Geslo** (vsaj 4 znaki)
3. Klikni **Registracija**. Po uspešni registraciji te sistem samodejno prijavi in preusmeri na nadzorno ploščo.

### 2.5.2 Pridobivanje admin pravic

Privzeto je novo registrirani uporabnik **navaden** (`isAdmin: false`). Admin pravice ne dobi prek UI — to mora narediti že obstoječi admin **ali** ročno prek baze.

#### Možnost A: prek namizne aplikacije (priporočeno za hitri setup)

1. Zaženi namizno aplikacijo (`./gradlew :composeApp:run`).
2. V stranskem meniju izberi **Upravljaj podatke → Uporabniki**.
3. Najdi svojega uporabnika, klikni **Uredi**.
4. V urejevalnem oknu označi **Administrator** in shrani.

#### Možnost B: direktno prek MongoDB shell

```bash
mongosh "mongodb://localhost:27017/virtual_estate"
> db.users.updateOne({ email: "tvoj@email.com" }, { $set: { isAdmin: true } })
```

### 2.5.3 Prijava

1. Odpri http://localhost:5173/login
2. Vnesi e-pošto in geslo
3. Klikni **Prijava**

Po uspešni prijavi se v zgornjem desnem kotu pojavi tvoje ime z zelenim pikastim indikatorjem (admin) ali brez (navaden uporabnik). Če si admin, se v navigaciji pojavi povezava **Admin**, ki vodi do upravitelja nepremičnin.

### 2.5.4 Trajnost prijave

Prijavni žeton (JWT) je shranjen v `localStorage` brskalnika in velja **24 ur**. Po preteku se moraš ponovno prijaviti. Z gumbom **Odjava** lahko sejo ročno končaš.

## 2.6 Preverjanje, da vse deluje

| Korak | Pričakovani izid |
|---|---|
| Odpri http://localhost:5173 | Pojavi se nadzorna plošča "Nepremičnine" z zemljevidom |
| Klikni **Filtri → Iskanje po opisu** in vnesi nekaj | Lista in zemljevid se osvežita |
| Odpri zemljevid, klikni krog/poligon orodje | Riseš lahko območje; po zaključku se rezultati filtrirajo |
| V admin vmesniku ustvari novo nepremičnino | V nadzorni plošči se brez osvežitve pojavi novi marker + toast obvestilo |
| V namizni aplikaciji klikni **Pridobi podatke** v zavihku **Pridobi s spleta** | Po nekaj sekundah se naloži seznam scrape-anih oglasov |
| V generatorju nastavi *Število zapisov: 5* in klikni **Generiraj** | Pojavi se 5 namišljenih nepremičnin pripravljenih za pošiljanje |

Če katerikoli izmed teh korakov ne deluje, preveri:

- **Konzolo brskalnika** (F12) za morebitne CORS ali 401 napake
- **Terminal z WebService** za napake povezave na bazo
- **MongoDB** je dejansko zagnan in dosegljiv

## 2.7 Pogosta vprašanja

**V: Strežnik ne najde MongoDB.**  
O: Preveri `DATABASE_URL` v `WebService/.env`. Privzeto pričakuje `mongodb://127.0.0.1:27017/virtual_estate`.

**V: Spletni vmesnik javi "Network Error" pri prijavi.**  
O: WebService verjetno ni zagnan, ali je na drugem portu. Preveri konzolo brskalnika za URL, ki ga axios kliče.

**V: Po prijavi me prekine k Login strani.**  
O: JWT žeton ni veljaven (pretekel je, ali se je `JWT_SECRET` spremenil). Odpri F12 → Application → Local Storage → izbriši `token` in se ponovno prijavi.

**V: Realnočasovne posodobitve ne pridejo.**  
O: WebSocket povezava je morda blokirana. V Vite proxy je `/socket.io` že pravilno usmerjen, ampak če uporabljaš obratno proxy (nginx ipd.), poskrbi, da je `Upgrade` glava ohranjena.
