# 4. Dokumentacija izvedenih lastnosti (feature)

Stran vsebuje podrobne opise **ključnih implementiranih funkcionalnosti** projekta VirtualEstate. Vsaka lastnost je dokumentirana po standardni strukturi:

- **Namen implementacije** — kateri problem rešitev rešuje in zakaj smo se odločili za to funkcionalnost
- **Način implementacije** — tehnično ozadje, knjižnice in arhitekturne odločitve
- **Način uporabe** — kako uporabnik aktivira oz. izkoristi funkcionalnost

Funkcionalnosti so urejene po Jira sledilnih oznakah (SCRUM-XX) oziroma po pomembnosti.

---

## Lastnost 1 — JWT avtentikacija in administrativna vloga (SCRUM-46)

### Namen implementacije
Sistem mora ločiti **navadne uporabnike** (lahko gledajo podatke) od **administratorjev** (lahko spreminjajo podatke). Implementacija JWT žetonov omogoča stateless avtentikacijo, primerno za horizontalno skaliranje, in se izogne zapletenosti server-side sej.

### Način implementacije

**Backend:**
- **Mongoose hook**: gesla so pred shranjevanjem hash-irana z `bcrypt` (cost factor 10) v `UserModel.js:12-21`.
- **Prijava** (`UserController.login`): preveri geslo prek `bcrypt.compare`; če uspe, podpiše JWT s payload-om `{ id, email, isAdmin }`, algoritem **HS256**, veljavnost **24 ur**.
- **Auth middleware** (`authMiddleware.js`): bere `Authorization: Bearer <token>` glavo, verificira podpis in vstavi `req.user` v Express request.
- **Admin middleware** (`adminMiddleware.js`): preveri `req.user.isAdmin === true`; sicer vrne 403.
- **AdminOrSelf middleware** (`adminOrSelfMiddleware.js`): dovoli dostop, če je uporabnik admin ALI če `:id` v URL-ju ustreza `req.user.id`.
- **Rate-limit** na `/api/users/login`: 10 zahtevkov / 15 min prek `express-rate-limit`.

**Frontend:**
- `AuthContext.jsx` hrani `user` in `token` v React state-u; persistira v `localStorage`.
- `axios.interceptors.request.use` avtomatsko doda `Authorization: Bearer <token>` v vsak zahtevek.
- `axios.interceptors.response.use` na 401 odgovoru počisti `localStorage` in preusmeri uporabnika na prijavo.
- `PrivateRoute` komponenta zaščiti rute (`/admin`) in opcijsko zahteva admin vlogo.

### Način uporabe
1. **Registracija**: nov uporabnik gre na `/register`, izpolni obrazec; po uspehu se samodejno prijavi.
2. **Prijava**: obstoječi uporabnik gre na `/login`. Po prijavi se v Navbar pojavi *user-chip* z imenom in (če admin) zelenim pikčastim indikatorjem.
3. **Admin vmesnik** je dostopen samo prijavljenim adminom prek povezave **Admin** v navigaciji ali direktno na `/admin`.
4. **Odjava**: gumb **Odjava** v Navbar počisti `localStorage` in stanje.

---

## Lastnost 2 — Realnočasovne posodobitve prek WebSocket (SCRUM-40)

### Namen implementacije
Ko admin doda, posodobi ali izbriše nepremičnino, mora to vsak prijavljeni uporabnik takoj videti — brez ročnega osveževanja. Klasično polling rešitev bi bilo neučinkovito; WebSocket ponuja persistentno dvosmerno povezavo.

### Način implementacije

**Backend** (`WebService/bin/www`):
- Inicializira `socket.io` server vezno na isti HTTP server kot Express.
- CORS dovoljuje vse izvore (development setup; v produkciji bi omejili).
- Express middleware `req.io = app.get('socketio')` izpostavi `io` instance vsakemu requestu.

**Emisija dogodkov** (v `PropertyController.js`):
```js
// po uspešnem create
if (req.io) req.io.emit('propertyCreated', saved);
// po uspešnem update
if (req.io) req.io.emit('propertyUpdated', saved);
// po uspešnem delete
if (req.io && property) req.io.emit('propertyDeleted', { _id: property._id });
```

**Frontend** (`WebClient/src/api/socket.js`):
- Vzpostavi `io(SOCKET_URL, { autoConnect: true, transports: ['websocket', 'polling'] })`.
- `Dashboard.jsx` v `useEffect` prijavi tri listenerje, ki posodobijo lokalni `properties` state in sprožijo *toast* obvestilo:

```js
socket.on('propertyCreated', p => {
  setProperties(prev => [p, ...prev]);
  showToast(`Nova nepremičnina: ${p.propertyType} v ${p.city}`, 'success');
});
```

### Način uporabe
Ko uporabnik odpre spletni vmesnik, se Socket.IO klient samodejno poveže. Vsaka sprememba (iz admin vmesnika ali namizne aplikacije) sproži:

1. **Posodobitev seznama** brez refresha.
2. **Posodobitev zemljevida** (marker se doda/posodobi/izbriše).
3. **Toast obvestilo** v desnem spodnjem kotu:
   - **Zeleno** za *create* (uspeh)
   - **Modro** za *update* (informacija)
   - **Oranžno** za *delete* (opozorilo)

Vsak toast ima ikono in 2.5–3.5 sekundno trajanje.

---

## Lastnost 3 — Geoprostorske poizvedbe in območni filter na zemljevidu (SCRUM-70)

### Namen implementacije
Iskanje "nepremičnine v polmeru 2 km od te točke" ali "nepremičnine znotraj območja, ki ga rišem" je naravnejši uporabniški izziv kot vnašanje radija v polje. Implementacija združuje **MongoDB geoindeks** (na backendu) z **Leaflet Draw** (na frontendu).

### Način implementacije

**Backend** (`PropertyController.js`):
- Schema ima `coordinates: { type: 'Point', coordinates: [lng, lat] }` z `2dsphere` indeksom (`PropertyModel.js:21`).
- `buildGeoFilter(query)` funkcija ([PropertyController.js:36-78](https://github.com/andrej-nemanic/VirtualEstate_Project/blob/develop/WebService/controllers/PropertyController.js#L36)) prepozna tri query parametre:
  - `bbox=minLng,minLat,maxLng,maxLat` → 4-točkovni `$geoWithin Polygon`
  - `polygon=lng,lat;lng,lat;…` → poljubni `$geoWithin Polygon` (avtomatsko zapre)
  - `near=lng,lat,radius_v_metrih` → `$geoWithin $centerSphere` (radius v radianih)
- Endpoint `GET /api/properties/search?lat&lng&distance` ohranja stari `$near` operator za preprosto iskanje po polmeru.

**Frontend** (`WebClient/src/components/PropertyMap.jsx`):
- Uporablja `leaflet-draw` (CDN CSS + import JS) za risarsko orodno vrstico.
- `L.Control.Draw` konfiguriran z dovoljenima oblikama: `polygon`, `circle`. Pravokotnik (`rectangle: false`), marker, polyline in circlemarker so onemogočeni.
- Po dogodku `L.Draw.Event.CREATED`:
  - Poligon → `getLatLngs()` → niz `lng,lat` parov ločenih z `;`
  - Krog → središče + radij v metrih → `near`
- Callback `onAreaSelected({ type, value })` posodobi URL parameter prek `useSearchParams` v `Dashboard.jsx`.
- Toast obvestilo z ikono *map-pin* in tipom oblike pojasni uporabniku, da je filter aktiven.

**Dark mode podpora**:
- `leaflet-draw` orodna vrstica v dark mode dobi `filter: invert(1) hue-rotate(180deg)`, kar pravilno obrne ikone in ozadje.
- `react-leaflet-cluster` markerji ostanejo nespremenjeni (njihove barve so kustomizirane prek `divIcon`).

### Način uporabe
1. Na nadzorni plošči poiščite orodno vrstico v zgornjem desnem kotu zemljevida (2 ikoni).
2. Izberite obliko, kliknite in povlecite za risanje.
3. Po končanem risanju se URL takoj posodobi (`?near=14.5,46.1,2000`) in seznam se filtrira.
4. Za brisanje: gumb **Počisti območje** v modrem obvestilu ali ikona smetnjaka v orodni vrstici.

---

## Lastnost 4 — Iskanje po opisu z varno regex obravnavo

### Namen implementacije
Uporabnik želi tipično iskati po besedah kot "balkon", "garaža", "klet" v opisu oglasa. MongoDB `$regex` je naravno orodje za to, ampak surov user input v regex je **klasični vektor napada** (ReDoS, zaobid filtra). Rešitev je escape meta-znakov.

### Način implementacije

**Backend** (`PropertyController.js`):
```js
function escapeRegex(value) {
    return String(value).replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

function buildListFilter(query) {
    const filter = {};
    if (query.description)
        filter.description = { $regex: escapeRegex(query.description), $options: 'i' };
    // ... isto za city, region, propertyType, source
}
```

Escape funkcija:
- Pokrije vseh 14 regex meta-znakov: `. * + ? ^ $ { } ( ) | [ ] \`
- Uporabljena na **vseh** regex filtrih (ne samo opisu), ker so iste težave drugje (npr. mesto z imenom `St. Peter`).
- Case-insensitive (`$options: 'i'`).

**Frontend** (`PropertyFilters.jsx`):
- Iskalno polje uporablja SVG ikono lupe (ne emoji, da je usklajeno z ostalo ikonografijo).
- Input je v `<div className="search-field">` z absolutno pozicionirano ikono in left-padding na input.

### Način uporabe
1. V iskalno polje na nadzorni plošči vnesite besedo (npr. `balkon`).
2. URL se posodobi v `?description=balkon`.
3. Vmesnik prikaže samo nepremičnine, ki imajo `balkon` (case-insensitive) v `description` polju.
4. Kombinacija več filtrov: vse so v AND povezavi (`description=balkon&city=Maribor&maxPrice=200000`).
5. Vsi simboli kot `(`, `*`, `?` v iskalnem nizu so varno obravnavani — sistem ne crasha in ne odpre varnostne ranljivosti.

---

## Lastnost 5 — Razčlenjevanje podatkov iz spletnih virov (SCRUM-46, SCRUM-69)

### Namen implementacije
Ročno vnašanje nepremičnin v bazo bi bilo prepočasno. Sistem mora samodejno pridobiti aktualne oglase iz slovenskih nepremičninskih portalov.

### Način implementacije

**Tehnologija**: **Ksoup** (Kotlin port knjižnice Jsoup) za HTML parsing.

**WebScraper.kt** (`DesktopApplication/composeApp/.../WebScraper.kt`):
- Funkcija `scrapeNepremicnina()` izvede `Ksoup.parseGetRequestBlocking` na `https://nepremicnina.si/nepremicnine`.
- CSS selektor `div.pzl-item.list` izbere posamezne oglase.
- Za vsak oglas razčleni:
  - **Lokacija** iz `img.alt` atributa (format `Lokacija: Regija, Mesto, Naselje`)
  - **Tip** iz `div.about h2` (prvi del pred vejico)
  - **Velikost** iz `div.about h3` z regex `(\d+(?:[.,]\d+)?)\s*m`
  - **Cena** iz `div.price` z `parseDouble` (čisti €, presledke, podpičja)
  - **Ponudba** iz `div.badge` (besedilo ali class `to-sell`/`to-rent`)
  - **Opis** iz `div.description > div`
  - **Slika** iz `img.pzl-gallery-item.src`
  - **Povezava** iz `a.about.href`
- `scrape24Nep()` dela enako za `https://24nep.si/oglasi` z drugačnimi CSS selektorji.
- `scrapeAll()` združi rezultate obeh portalov.

**WebSourcesScreen.kt**:
- Compose UI s tabelo, paginacijo (10/stran), sortiranjem, iskanjem in checkbox-i.
- Klik **Pridobi podatke** → `Dispatchers.IO` korutina → `WebScraper.scrapeAll()` → posodobi state.
- Klik **Pošlji v bazo (N)** → `Async.ingestAll` → zaporedni `POST /api/properties/ingest`.

**Backend ingest endpoint**:
- `POST /api/properties/ingest` v `PropertyRoutes.js` je **brez** JWT zaščite (za poenostavljen development; v produkciji bi dodali).
- `PropertyController.ingestCreate` shrani zapis, geokodira manjkajoče koordinate prek Nominatim, broadcasta WebSocket event.

### Način uporabe
1. V namizni aplikaciji odprite **Pridobi s spleta**.
2. Kliknite **Pridobi podatke**. Po nekaj sekundah se naloži 20–60 zapisov.
3. Po želji filtrirajte, sortirajte, označite.
4. Kliknite **Pošlji v bazo**.
5. Preverite v spletnem vmesniku — zapisi se pojavijo z oznako vira (`badge: nepremicnina.si` ali `24nep.si`).

---

## Lastnost 6 — Generator namišljenih podatkov

### Namen implementacije
Za demonstracije, učne primere in testiranje je nujno hitro pripraviti realne podatke v praznem sistemu. Namesto kompleksne knjižnice (`kotlin-faker`) smo implementirali generator z naborom 8 slovenskih krajev z znanimi koordinatami, kar zagotovi geografsko realističen dataset.

### Način implementacije

**DataGenerator.kt**:
```kotlin
private val locations = listOf(
    LocationOption("Osrednjeslovenska", "Ljubljana", 14.5058, 46.0569,
        listOf("Šiška", "Bežigrad", "Center", "Vič", "Moste")),
    LocationOption("Podravska", "Maribor", 15.6459, 46.5547,
        listOf("Tabor", "Melje", "Pobrežje", "Tezno", "Studenci")),
    // ... 6 more cities
)
```

Funkcija `generate(count, priceRange, sizeRange)`:
- Za vsako iteracijo naključno izbere lokacijo iz seznama.
- Dodeli **jitter** ±0.01° na koordinatah, da se markerji ne prekrivajo.
- Naključno izbere tip (`Stanovanje`, `Hiša`, `Vikend`, `Poslovni prostor`, `Garaža`, `Parcela`) in ponudbo (`Prodaja`, `Oddaja`).
- Cena in m² sta naključna v uporabnikovih razponih.

**GeneratorScreen.kt**:
- Compose UI s polji za parametre, gumboma **Generiraj** in **Pošlji v bazo (N)**.
- Po generiranju se zapisi pojavijo v isti tabelni komponenti kot scraping (skupna UX).
- Po pošiljanju se zapisi avtomatsko počistijo iz lokalnega state-a.

### Način uporabe
1. V namizni aplikaciji odprite **Generator podatkov**.
2. Nastavite parametre (npr. 50 zapisov, cena 50k–500k €, velikost 30–200 m²).
3. Kliknite **Generiraj** — zapisi se v trenutku pojavijo.
4. Po želji odznačite tiste, ki vam niso všeč.
5. Kliknite **Pošlji v bazo**.

---

## Lastnost 7 — Vizualizacije: grafi in zemljevid s klastriranjem

### Namen implementacije
Številčne tabele so težko berljive. Grafi in zemljevid omogočajo hitri vizualni pregled trga: kateri tipi so najpogostejši, kakšne so povprečne cene, kje so nepremičnine geografsko skoncentrirane.

### Način implementacije

**Grafi** (`PropertyCharts.jsx` z **Recharts**):
- **Pie chart**: število nepremičnin po tipu. Vsak segment ima svojo barvo iz palete.
- **Bar chart**: povprečna cena po tipu. Vrednost je prikazana z `€` formatiranjem.
- **Scatter chart**: razmerje cena vs. velikost. Pomaga razbrati outlier oglase in trende.

Vsi grafi:
- Filtrirani po izbrani ponudbi (Prodaja / Oddaja) prek **segmented control**. Razlog: cene prodaje (100k+) in oddaje (500–1500 €/mes) so v različnih redih velikosti.
- Vsak graf je samostojno togglable prek **chip-list** (uporabnik lahko skrije tiste, ki ga ne zanimajo).
- Med nalaganjem se prikažejo *skeleton* placeholderji (gradient shimmer).

**Zemljevid** (`PropertyMap.jsx` z **Leaflet** + **react-leaflet** + **react-leaflet-cluster**):
- **Tile layer**:
  - Light mode: standardni OpenStreetMap (`tile.openstreetmap.org`)
  - Dark mode: CARTO Dark (`basemaps.cartocdn.com/dark_all`)
- **Marker clustering**: pri večjem številu zapisov se markerji avtomatsko združujejo v cluster-je. Klik na cluster zoom-a in razdeli.
- **Custom markerji**: vsak tip nepremičnine ima svojo barvo (`divIcon` z inline CSS).
- **Popup**: ob kliku na marker prikaže sliko (če obstaja), tip, ponudbo, lokacijo, ceno, m² in povezavo do oglasa.
- **FitBounds**: ko uporabnik prvič odpre stran ali počisti območni filter, se zemljevid avtomatsko zoom-a, da prikaže vse markerje.

### Način uporabe
1. **Grafi**: kliknite gumb **Grafi in statistike** na nadzorni plošči. Razširi se panel z tremi grafi.
2. Preklopite med **Prodaja** in **Oddaja** s segmented control-om.
3. Skrijte ali pokažite posamezne grafe s klikom na chip.
4. **Zemljevid**: scroll za zoom, drag za premik, klik na marker za popup. Cluster se razdeli ob zadostnem zoom-u.

---

## Lastnost 8 — Dark/light mode in odzivni vmesnik

### Namen implementacije
Sodobni vmesniki morajo prilagajati izgled tako uporabnikovim preferencam (dark/light), kot tudi različnim velikostim ekrana. Cilj je bil **brezhibna izkušnja na vseh napravah**, od mobilnega do širokega monitorja.

### Način implementacije

**Dark/light mode** (`ThemeContext.jsx` + `index.css`):
- CSS custom properties (CSS variables) so definirane na `:root` (light) in `[data-theme="dark"]` (dark).
- Vsak `--bg`, `--surface`, `--text`, `--primary` itd. ima par za oba načina.
- `ThemeContext` shrani izbiro v `localStorage` in postavi `data-theme` atribut na `<html>`.
- Toggle gumb je v Navbar (ikona sonca/lune).
- Vsi grafi, zemljevid in skeleton loaderji avtomatsko sledijo temi prek CSS spremenljivk.

**Mobile-first responsive** (`index.css`):
- **Navbar**: na mobilnih napravah se zloži v burger meni s polno-širokimi gumbi (touch-target ≥ 40 px).
- **Filter toolbar**: search polje 100% širina, select in gumbi se enakomerno razdelijo.
- **Property cards**: 3 stolpci na desktopu, 2 stolpca na tabletu, 1 stolpec na mobilu.
- **Pagination**: gumbi se enakomerno razdelijo (50/50), info gre nad njiju.
- **Stats-pill** ima skeleton placeholder med nalaganjem.
- **Empty state**: SVG ikona (search-x) z naslovom, opisom in *Počisti filtre* CTA-jem.
- **Toast**: variante (success/info/warning) s SVG ikono in barvnim levo-stranskim borderjem.

**A11y dotiki**:
- `aria-label` na ikoničnih gumbih (pagination, navbar toggle)
- `aria-live="polite"` na dinamičnih elementih (pagination info, toast)
- `aria-sort` na sortable th elementih v admin tabeli
- `aria-expanded` na collapsible expandable elementih

### Način uporabe
1. **Preklop teme**: kliknite ikono sonca/lune v desnem zgornjem kotu navigacije.
2. Izbira se shrani; pri naslednjem obisku je tema avtomatsko upoštevana.
3. **Mobile**: odprite spletni vmesnik na telefonu ali zmanjšajte okno brskalnika pod 768 px — navigacija postane burger meni, vsi elementi se zložijo v en stolpec.

---

## Sklep

Zgornjih 8 funkcionalnosti pokriva vse **obvezne** in **razširjene** zahteve iz projektnih specifikacij. Vse so popolnoma delujoče, pokrite z dokumentacijo in dostopne prek uporabniškega vmesnika (spletnega ali namiznega) brez potrebe po dodatnem CLI ali ročnem urejanju datotek. Za vsako lastnost obstaja Jira sledilna oznaka (SCRUM-XX), prek katere je mogoče pregledati zgodovino sprememb in pull request-ov.

Funkcionalnosti, ki so bile **odložene** ali **izven obsega** projekta:

- **Cron auto-scraping** — strežniško periodično razčlenjevanje. Odloženo za naslednji semester.
- **Pozabljeno geslo** — reset prek e-pošte. Izven obsega prve verzije.
- **Animirane vizualizacije skozi čas** — predvidoma povezano z `priceHistory` modelom v naslednjem semestru.
- **Več slik na nepremičnino** — model trenutno hrani le `imageUrl: String`.
