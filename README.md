# Izbori Srbija — interaktivna mapa

Choropleth mapa rezultata parlamentarnih izbora u Srbiji (2000–2023), sa
biranjem teritorijalnog nivoa (opština → grad → okrug → region → pokrajina →
republika), godine i liste/kandidata. Radi kao samostalna statička stranica
(`index.html`) — Leaflet učitava granice i rezultate direktno sa ovog
GitHub repoa preko `raw.githubusercontent.com`, bez servera i bez build koraka.

**Otvori:** preuzmi `index.html` i otvori ga u browseru (radi i lokalno i
sa GitHub Pages), ili ga hostuj bilo gde — sve što mu treba su fajlovi
navedeni ispod, na fiksnim URL-ovima ovog repoa.

## Izvori podataka

- **Rezultati izbora** — Republički zavod za statistiku (RZS),
  `data.stat.gov.rs`, JSON export po izbornom ciklusu (originalni fajlovi u
  folderima `Izbori za narodne poslanike .../Result-*.json`).
- **Geografske granice** — GeoSrbija / RGZ (Republički geodetski zavod),
  `download.geosrbija.rs`, sloj "Registar prostornih jedinica", geojson
  export (originalni fajlovi `Opstina.geojson`, `Okrug.geojson`,
  `Grad.geojson`, `Republika.geojson`, `NSTJ1/2/3.geojson`).
- **Šifarnici** (matični brojevi opština/gradova, nazivi, uprava okruzi) —
  RZS, `statisticki_sifarnik-1.xls`, `sifarnikgradovi-opstine-tekucestanje.xlsx`,
  `nstj-kodovi-_sifre.xls`.

## Struktura repoa

```
index.html                    stranica sa mapom (jedini fajl potreban za rad)

geo_opstina.geojson           granice opština i gradskih opština (Beograd, Niš), 185 jedinica
geo_grad.geojson              granice "gradova" (status grad), 29 jedinica
geo_okrug.geojson             granice upravnih okruga, 30 jedinica
geo_region.geojson            granice 5 statističkih regiona (NSTJ2)
geo_pokrajina.geojson         granice 3 "pokrajine" (Vojvodina / Centralna Srbija / Kosovo i Metohija)
geo_republika.geojson         granica cele Republike Srbije

election_2000.json            obrađeni rezultati po ciklusu, svi dostupni nivoi
election_2003.json
election_2007.json
election_2008.json
election_2012.json
election_2014.json
election_2016.json
election_2020.json
election_2023.json            samo region/pokrajina/republika — vidi "Ograničenja"

Izbori za narodne poslanike .../Result-*.json   originalni RZS export (sirovi podaci, arhiva)
Opstina.geojson, Okrug.geojson, ...             originalni RGZ export (sirovi podaci, arhiva)
naselje_1.geojson … naselje_5.geojson           granice naseljenih mesta (arhiva, mapa ih ne koristi — vidi ispod)
sifarnik*.xls(x)                                šifarnici korišćeni za obradu
```

Fajlovi bez prefiksa `geo_`/`election_` su **sirovi izvorni podaci** —
arhiva, mapa ih ne čita direktno. `geo_*` i `election_*` su obrađeni,
web-spremni derivati (reprojektovani u WGS84, pojednostavljena geometrija,
imena/šifre usklađene preko šifarnika). Menjaju se samo kad se menja
metod obrade — ne generišu se iznova pri svakoj poseti stranici.

## Format `election_*.json`

```json
{
  "year": "2020",
  "label": "Izbori za narodne poslanike ... 21. jun i 1. jul 2020. godine",
  "levels": {
    "opstina": {
      "70661": { "n": "Krupanj", "t": 8421, "c": { "ALEKSANDAR VUČIĆ – ...": 5210, "...": 900 } }
    },
    "grad": { "...": "..." },
    "okrug": { "...": "..." },
    "region": { "...": "..." },
    "pokrajina": { "...": "..." },
    "republika": { "...": "..." }
  }
}
```

`n` = naziv jedinice, `t` = ukupno glasova u jedinici, `c` = glasovi po
listi/kandidatu. Procenat se računa u browseru (`glasovi / t`), ne čuva se
unapred izračunat.

Ključ u `levels.<nivo>` je šifra jedinice i mora se poklapati sa
`properties.kod` u odgovarajućem `geo_<nivo>.geojson`-u — to je jedina veza
između geometrije i podataka.

## Kako su nivoi izvedeni jedan iz drugog

RZS-ovi originalni fajlovi (2000–2020) sadrže tekstualni naziv teritorijalne
jedinice (`nTer`) na više nivoa pomešano u istoj listi — opštine, upravni
okruzi, pokrajine i republika, sve kao redovi u istom fajlu, raspoznati
samo po nazivu. Obrada radi ovako:

1. Normalizuje se svaki `nTer` (ćirilica→latinica, uklanjanje razmaka/znakova).
2. Poredi se sa šifarnikom opština/gradova → dodeljuje se matični broj
   (opština/gradska opština nivo).
3. Nazivi koji sadrže "upravni okrug" upoređuju se sa `Okrug.geojson`
   svojstvima → okrug nivo.
4. Nazivi "Vojvodina" / "Centralna Srbija" / "Kosovo i Metohija" mapiraju
   se direktno → pokrajina nivo.
5. "Grad" nivo se ne čita direktno iz RZS fajla nego **izvodi** iz opštinskog
   nivoa: za Beograd i Niš se sabiraju glasovi njihovih gradskih opština, za
   ostale "gradove" (Novi Sad, Kragujevac, Šabac, Čačak...) grad = ista
   jedinica kao opština (imaju isti matični broj).
6. Region (NSTJ2, 5 regiona) se ne objavljuje u RZS fajlu za 2000–2020 —
   izveden je **geometrijski**: centroid svakog okruga testiran je da li
   upada u koji od 5 poligona iz `NSTJ2.geojson`, pa se glasovi okruga
   sabiraju na taj region.
7. Tri naselja koja se u pojedinim ciklusima javljaju kao posebni redovi
   (Kostolac, Sevojno, Vranjska Banja) pripojena su matičnoj opštini
   (Požarevac, Užice, Vranje) — ona nemaju sopstvenu poligon granicu u
   `Opstina.geojson`.
8. Redovi koji nisu geografski smestivi (Inostranstvo, Zavodi za izvršenje
   zavodskih sankcija, Interno raseljeni birači) se ne uključuju ni u jedan
   nivo.

Ceo postupak (2–8) je deterministički kod, ne ručna lista — ponovljiv je za
svaku narednu godinu čim se doda njen `Result-*.json` u istom (starom)
formatu.

## Ograničenja

- **2023. ciklus nema opštinski/gradski/okružni nivo.** RZS je za taj
  izborni ciklus promenio format objavljivanja — `Result-07021101-*.json`
  sadrži glasove samo do nivoa 5 regiona (NSTJ2) i republike, opštinski
  detalj nije objavljen u ovom exportu. Dropdown za nivo se u stranici
  automatski suzi kad je izabrana 2023.
- **Nema nivoa naseljenog mesta.** Parlamentarni izbori u Srbiji se biraju
  kao jedna nacionalna izborna jedinica — RZS ne objavljuje rezultate po
  naseljenom mestu (samo do opštine), tako da `naselje_*.geojson` slojevi
  nisu iskorišćeni za mapu iako postoje u repou.
- Geometrija je pojednostavljena (Douglas-Peucker, tolerancija ~100–300m u
  zavisnosti od nivoa) radi veličine fajla na mobilnom — nije pogodna za
  precizna geodetska merenja, samo za vizuelni prikaz.

## Dodavanje nove godine (kad izađe novi RZS export)

1. Ubaci originalni `Result-*.json` u odgovarajući folder ciklusa.
2. Ako fajl ima `IDTer` polje (novi format, kao 2023) → sadrži samo
   region/republika nivo, obradi po uzoru na `election_2023.json`.
3. Ako fajl ima samo `nTer` tekstualna polja (stari format, kao 2000–2020)
   → prođi kroz isti pipeline (koraci 1–8 iznad) da dobiješ pun set nivoa.
4. Dodaj godinu u `YEARS` niz u `index.html` (i u `LEVELS_BY_YEAR` ako
   nedostaje neki nivo).

## Licenca / izvor podataka

Podaci: RZS (data.stat.gov.rs) i RGZ/GeoSrbija (download.geosrbija.rs),
javno dostupni državni izvori. Kod stranice je slobodan za izmenu i
ponovnu upotrebu.
