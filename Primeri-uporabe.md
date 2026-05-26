# 3. Ključni primeri uporabe

Stran vsebuje **pet najpogostejših scenarijev**, kako uporabnik dela z rešitvijo VirtualEstate. Vsak scenarij sledi enotni strukturi: **Kaj uporabnik želi narediti**, **Predpogoji**, **Koraki izvedbe** in **Rezultat**. Scenariji so urejeni od najpogostejšega (pregled in iskanje) do najnaprednejšega (generiranje testnih podatkov).

---

## Primer 1 — Pregled nepremičnin in iskanje s filtri

### Kaj uporabnik želi narediti
Iskalec stanovanja želi najti hišo ali stanovanje na določenem območju z določenimi lastnostmi (npr. cena do 200.000 €, vsaj 80 m², beseda "balkon" v opisu).

### Predpogoji
- WebService in WebClient sta zagnana.
- V bazi je vsaj nekaj nepremičnin (sicer najprej izvedi Primer 4 ali 5).

### Koraki izvedbe

1. **Odpri spletni vmesnik** na http://localhost:5173. Pojavi se stran **Nepremičnine** z zemljevidom in seznamom.

2. **Vnesi besedo v iskalno polje** (`Iskanje po opisu`). Vtipkaj npr. `balkon`. Seznam in zemljevid se osvežita v realnem času (URL se posodobi z `?description=balkon`).

3. **Odpri razširjene filtre** s klikom na gumb **Filtri**. Pojavi se razširjeni panel.

4. **Nastavi filtre**:
   - **Tip ponudbe**: `Prodaja`
   - **Regija**: `Podravska`
   - **Mesto / občina**: `Maribor`
   - **Maks. cena (€)**: `200000`
   - **Min. velikost (m²)**: `80`

5. **Aktivni filtri** se pojavijo kot odstranljive *pille* nad seznamom. Vsako lahko odstraniš s klikom na `×`.

6. **Pregled rezultatov**:
   - **Število zadetkov** je vidno v desnem zgornjem kotu (npr. `12 zadetkov`).
   - **Zemljevid** prikaže markerje samo za zadetke.
   - **Seznam** je razdeljen v 2 stolpca × 3 vrstice (6 zadetkov/stran); na dnu so paginacijski gumbi.

7. **Klikni na nepremičnino** v seznamu (če ima `propertyLink`, te odpre originalen oglas v novem zavihku).

8. **Deljenje iskanja**: kopiraj URL iz brskalnika (`http://localhost:5173/?description=balkon&offerType=Prodaja&...`) in ga pošlji prijatelju. Pri njem se odpre identično stanje filtrov.

### Rezultat
Uporabnik vidi seznam in zemljevid samo s tistimi nepremičninami, ki ustrezajo njegovim kriterijem. URL je deljiv. Po želji lahko klikne **Počisti** za reset.

---

## Primer 2 — Iskanje nepremičnin na izbranem območju zemljevida

### Kaj uporabnik želi narediti
Investitor želi videti **vse nepremičnine v določenem kraju** (npr. v polmeru 2 km okoli specifične lokacije), ne glede na to, ali pozna ime mesta.

### Predpogoji
- WebClient odprt na strani **Nepremičnine**.
- V bazi so nepremičnine z veljavnimi koordinatami.

### Koraki izvedbe

1. **Najdi orodno vrstico za risanje** v zgornjem desnem kotu zemljevida (tri ikone: pravokotnik, poligon, krog).

2. **Izberi obliko**:
   - **Pravokotnik** — klikni in povleci za izbor pravokotnega območja.
   - **Poligon** — klikaj točke; dvojni klik zaključi poligon. Idealno za nepravilne meje (npr. obliko mestnega okrožja).
   - **Krog** — klikni središče, povleci za določitev polmera. Najpogostejši za "iščem v polmeru X km od te točke".

3. **Po zaključku risanja** se sproži filter. URL se posodobi z eno izmed naslednjih parametričnih vrednosti:
   - `?bbox=lng1,lat1,lng2,lat2` (pravokotnik)
   - `?polygon=lng1,lat1;lng2,lat2;…` (poligon)
   - `?near=lng,lat,radius_v_metrih` (krog)

4. **Vmesnik** prikaže modro obvestilo:
   > 📍 Aktivno območje na zemljevidu (krog) — prikazani so samo zadetki znotraj.

5. **Pregled v seznamu** — paginiran seznam pod zemljevidom prikaže samo zadetke v izbrani regiji.

6. **Kombinacija z drugimi filtri**: medtem ko je območni filter aktiven, lahko še naprej tipkash v iskalno polje ali odpiraš razširjene filtre — vsi se kombinirajo z `AND` semantiko.

7. **Brisanje območja**:
   - Klikni **Počisti območje** v modrem obvestilu, ALI
   - Klikni ikono za brisanje v orodni vrstici zemljevida (smetnjak).

### Rezultat
Uporabnik vidi samo nepremičnine znotraj poljubno določene geografske oblike. Filter je trajno shranjen v URL-ju — strani lahko deli ali doda med priljubljene. Backend uporabi MongoDB `$geoWithin` (pravokotnik, poligon) ali `$centerSphere` (krog) za hitro filtriranje na 2dsphere indeksu.

---

## Primer 3 — Ročno dodajanje nepremičnine (administrator)

### Kaj uporabnik želi narediti
Administrator (npr. nepremičninska agencija) želi v sistem dodati nov oglas, ki ga nima na nobenem portalu (npr. zasebna prodaja).

### Predpogoji
- Uporabnik je **prijavljen** kot admin (glej poglavje 2.5.2 v [Navodilih za namestitev](Namestitev-in-prijava)).

### Koraki izvedbe

1. **V zgornjem meniju klikni** povezavo **Admin**. Odpre se stran **Upravitelj nepremičnin**.

2. **Klikni gumb** **+ Nova nepremičnina**. Odpre se obrazec.

3. **Izpolni obvezna polja**:
   - **Regija**: `Osrednjeslovenska`
   - **Mesto / občina**: `Ljubljana`
   - **Naselje** (opcijsko): `Šiška`
   - **Tip ponudbe**: `Prodaja`
   - **Vrsta nepremičnine**: `Stanovanje`
   - **Vir podatka**: `ročno` (privzeto)
   - **Velikost (m²)**: `65`
   - **Cena (€)**: `185000`
   - **Opis** (opcijsko): `Lepo opremljeno 2-sobno stanovanje z balkonom in parkirnim mestom.`
   - **Povezava do oglasa** (opcijsko): `https://primer.si/oglas/123`
   - **URL slike** (opcijsko): `https://images.com/foto.jpg`

4. **Geo koordinate** so opcijske. Če jih pustiš prazne, backend bo samodejno geokodiral naslov prek Nominatim API-ja (regija + mesto + naselje). Če imaš točne koordinate, jih vnesi:
   - **Geo. dolžina (lng)**: `14.5058`
   - **Geo. širina (lat)**: `46.0569`

5. **Klikni** **Ustvari**.

6. **V trenutku** se v tabeli pod obrazcem pojavi nova vrstica. Hkrati se broadcast prek WebSocket-a sproži in vsi drugi aktivni odjemalci (vključno z navadnimi uporabniki na nadzorni plošči) takoj vidijo nov marker na zemljevidu in **toast obvestilo**:
   > ✓ Nova nepremičnina: Stanovanje v Ljubljana

7. **Urejanje** — klikni **Uredi** v vrstici. Odpre se isti obrazec, pred-izpolnjen z obstoječimi vrednostmi.

8. **Brisanje** — klikni **Briši**. Pojavi se potrditveno okno; po potrditvi se zapis izbriše.

### Rezultat
Nov zapis je v bazi (`POST /api/properties` z JWT žetonom + admin vlogo). Vsi odjemalci so v realnem času obveščeni. Zapis ima dodeljene koordinate (ročno ali samodejno geokodirane) in je takoj viden na zemljevidu.

---

## Primer 4 — Razčlenjevanje podatkov iz spletnih virov (namizna aplikacija)

### Kaj uporabnik želi narediti
Administrator želi v bazo uvoziti **več sto oglasov** s slovenskih nepremičninskih portalov, brez ročnega vnašanja.

### Predpogoji
- WebService je zagnan.
- Namizna aplikacija je zagnana (`./gradlew :composeApp:run`).
- Računalnik ima dostop do interneta.

### Koraki izvedbe

1. **V namizni aplikaciji** klikni v stranskem meniju **Pridobi s spleta**.

2. **Klikni** **Pridobi podatke**. Aplikacija v ozadju izvede zahtevek na:
   - https://nepremicnina.si/nepremicnine
   - https://24nep.si/oglasi

   in razčleni HTML strukturo (uporablja knjižnico **Ksoup**).

3. **Po nekaj sekundah** se v tabeli pojavi seznam **vseh razčlenjenih oglasov** (običajno 30–60 zapisov). Prikazani so atributi: lokacija (regija/mesto/naselje), tip, ponudba, m², cena, opis.

4. **Filtriraj rezultate** (preden jih pošlješ v bazo) — uporabi katerokoli kombinacijo:
   - **Iskanje** po besedi (regija, mesto, opis)
   - **Tip** (`Stanovanje`, `Hiša`, …)
   - **Ponudba** (`Prodaja` / `Oddaja`)
   - **Cena od / do** in **m² od / do**

5. **Sortiraj** s klikom na naslov stolpca (lokacija, tip, ponudba, m², cena). Drug klik obrne smer.

6. **Označi zapise** za pošiljanje:
   - **Posamično**: označi checkbox v vrstici.
   - **Vse**: označi checkbox v glavi tabele.

7. **Klikni** **Pošlji v bazo (N)** (gumb pokaže število izbranih). Aplikacija zaporedno pošlje vsak zapis prek `POST /api/properties/ingest`.

8. **Status sporočilo** se pojavi pod glavo (zeleno za uspeh, rdeče za napako):
   > Uspešno shranjenih 42 zapisov.

9. **Preveri rezultat** v spletnem vmesniku — novi zapisi se pojavijo v seznamu in na zemljevidu z oznako vira (`badge: nepremicnina.si` ali `24nep.si`).

### Rezultat
Baza je obogatena z aktualnimi oglasi iz dveh slovenskih portalov. Vsi imajo dodeljene koordinate (geokodirane na backend strani) in oznako vira, kar omogoča kasnejšo analizo. Razčlenjeni zapisi v namizni aplikaciji se po pošiljanju počistijo.

---

## Primer 5 — Generiranje namišljenih podatkov za testiranje

### Kaj uporabnik želi narediti
Razvijalec ali predavatelj želi v prazni bazi hitro generirati realno videti dataset za demonstracijo ali testiranje (npr. 50 nepremičnin v različnih slovenskih krajih).

### Predpogoji
- WebService je zagnan.
- Namizna aplikacija je zagnana.

### Koraki izvedbe

1. **V namizni aplikaciji** klikni v stranskem meniju **Generator podatkov**.

2. **Nastavi parametre**:
   - **Število zapisov**: `50`
   - **Cena min (€)**: `50000`
   - **Cena max (€)**: `500000`
   - **Velikost min (m²)**: `30`
   - **Velikost max (m²)**: `200`

3. **Klikni** **Generiraj**. Aplikacija v trenutku ustvari 50 zapisov:
   - Lokacija je naključno izbrana iz seznama 8 slovenskih krajev z znano regijo, mestom in koordinatami (Ljubljana, Maribor, Kranj, Škofja Loka, Celje, Koper, Murska Sobota, Nova Gorica).
   - Naselje je naključno izbrano iz nabora za vsako mesto (npr. Ljubljana: Šiška, Bežigrad, Center, Vič, Moste).
   - Koordinate dobijo majhno naključno *jitter* deviacijo (±0.01°), da se ne vsi markerji prekrivajo.
   - Tip ponudbe je naključno `Prodaja` ali `Oddaja`.
   - Vrsta nepremičnine je naključna iz 6 tipov (`Stanovanje`, `Hiša`, `Vikend`, `Poslovni prostor`, `Garaža`, `Parcela`).
   - Cena in m² sta naključna v okviru tvojih razponov.

4. **Pregled in filtriranje** — generirani zapisi se pojavijo v tabeli. Lahko jih filtriraš (po tipu, ponudbi) ali sortiraš (po regiji, ceni, m²).

5. **Označi**: privzeto so označeni vsi. Lahko odznačiš tiste, ki ti niso všeč (npr. če je cena nerealna za tip).

6. **Klikni** **Pošlji v bazo (N)**. Tako kot pri scrapingu se zapisi zaporedno shranijo prek `POST /api/properties/ingest`.

7. **Status**:
   > Uspešno shranjenih 50 zapisov.

8. **Preveri rezultat** v spletnem vmesniku — odpri http://localhost:5173. Zemljevid je zdaj poln markerjev po vsej Sloveniji, grafi prikazujejo distribucijo tipov in cen.

### Rezultat
Baza ima 50 realnih (a izmišljenih) zapisov, primernih za demonstracijo vseh funkcionalnosti spletnega vmesnika (grafi, zemljevid, filtri). Generiranje je deterministicno reproduktibilno (znotraj randoma) in v sekundah pripravi celoten dataset.

---

## Sklep

Pet zgornjih primerov pokriva celoten življenjski cikel uporabe sistema:

- **Primer 1 in 2**: tipično uporabniško izkušnjo iskalca.
- **Primer 3**: vsakodnevno delo administratorja.
- **Primer 4**: avtomatizacijo polnjenja baze iz realnih virov.
- **Primer 5**: pripravo testnih podatkov za demonstracije.

Vsi primeri uporabljajo iste REST endpointe in WebSocket dogodke, kar zagotavlja **konsistentno obnašanje** med spletnim in namiznim vmesnikom. Za podroben opis vsake funkcionalnosti glej [Izvedene lastnosti](Izvedene-lastnosti).
