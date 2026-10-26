# A türelmetlen gyógyszerkereső – hibariport

Repó: https://github.com/AndreaJulia24/umfst-bh08-gyogyszerkereso
Készítette: Czellecz Andrea-Julia, INFO-LM III.ev

## 1. hiba Versenyhelyzet (Race Condition) a hálózati válaszoknál
- **Hiba leírása és reprodukálása:** Gyorsan begépelve az „Algo” szót, a beviteli mezőben „Algo” szerepel, mégis az „A”-ra vonatkozó 5 találat marad a képernyőn a helyes 2 helyett.
- **Gyökérok:** A fakeFetch szimulációjában a rövidebb szövegek (pl. "A") lassabban válaszolnak. Ha 100 ms-onként gépelünk, ez történik: 
  Az "A" elindul 0 ms-nál (válasz: 1000 ms -> kész: 1000 ms-nál).
  Az "Algo" elindul 300 ms-nál (válasz: 250 ms -> kész: 550 ms-nál).
  Hiába az "Algo" az utolsó gépelés, hamarabb megérkezik, majd a legelső "A" kérés 1000 ms-nál befut, és felülírja a felületet az elavult adattal.
- **Javítás (commit link):** https://github.com/AndreaJulia24/umfst-bh08-gyogyszerkereso/commit/fe4c3e03ad9e4b1bf4b73e278115abffa7191b40
- **Hogyan tesztelted:** Konzolnaplózással ellenőriztem, hogy a gyorsan gépelt betűk lefutnak-e. A `keresettId !== utolsoKeres` feltétel megakadályozza, hogy az elavult kérés frissítse a DOM-ot.
- **Tanulság / érintett alapelv:** „A legutolsó kérés nyer” (Latest request wins). Egy kérésszámlálóval (ID) azonosítani kell, hogy melyik volt az utolsó releváns keresés.

## 2. hiba: Felesleges kérés-áradat minden billentyűleütésre (Hiányzó Debounce)
- **Hiba leírása és reprodukálása:** Minden egyes billentyűleütés azonnal elindít egy külön hálózati kérést. Ha gyorsan beírjuk, hogy "Algo", 4 külön kérés indul a szerver felé.
- **Gyökérok:** Az `input` eseménykezelőben nincs semmilyen késleltetés, így a `fakeFetch` azonnal lefut minden egyes karakternél.
- **Javítás (commit link):** https://github.com/AndreaJulia24/umfst-bh08-gyogyszerkereso/commit/cd818574eb668c218ec36bbfe2d6f8ceb5929a57
- **Hogyan tesztelted:** A konzolon (DevTools) figyeltem a kéréseket. A javítás után az „Algo” gyors legépelésekor 4 helyett már csak 1 kérés megy ki a szerverhez (miután megálltam a gépeléssel 300 ms-ra).
- **Tanulság / érintett alapelv:** „Debounce” (Késleltetés) alapelv. A hálózati forgalom és a szerver terhelésének csökkentése érdekében a gyorsan ismétlődő eseményeket össze kell vonni, és csak egy rövid szünet után szabad elküldeni a kérést.

## 3. hiba: Aszinkron hiba elkapása szinkron try/catch blokkal (Kifagyó felület)
- **Hiba leírása és reprodukálása:** Ha az „x” karaktert írjuk be, a felület örökre „Keresés…” állapotban ragad, a konzolban pedig `Uncaught (in promise) Error: Szerverhiba (503)` jelenik meg.
- **Gyökérok (eseményhurok, Promise-ok időbeli sorrendje):** A `fakeFetch` egy aszinkron hálózati kérést szimulál. A szinkron `try...catch` blokk azonnal, még az aktuális eseményhurok-ciklusban lefut. A hiba (Promise elutasítás / reject) viszont csak egy későbbi időpontban, a mikrotaszk sorból érkezik meg. Mire a hiba ténylegesen bekövetkezik, a `try...catch` már rég befejezte a futását, így a kivétel kezeletlenül marad (Uncaught), és a felület sem frissíti a DOM-ot.
- **Javítás (commit link):** https://github.com/AndreaJulia24/umfst-bh08-gyogyszerkereso/commit/5e3b2e634d05312dc8d733ca58b77a6c60fd54e5
- **Hogyan tesztelted:** Beírtam az „x”-et. A „Keresés…” felirat eltűnt, helyette megjelent a „Hiba: Szerverhiba (503)” szöveg a felületen, a konzol pedig teljesen tiszta maradt.
- **Tanulság / érintett alapelv:** Mindig kezeld a Promise elutasítását. Aszinkron kód hibáit a láncolt `.catch()` metódussal kell lekezelni (vagy `async/await` használata esetén a hozzá tartozó `try...catch`-csel).

## 4. hiba: Aszinkron ciklus kezelési hiba a forEach-ben (Mindig 0 összeg)
- **Hiba leírása és reprodukálása:** Sikeres keresés után a „Találatok listaáron összesen” mezőben mindig 0 lej marad az összeg, sosem frissül.
- **Gyökérok (eseményhurok, Promise-ok időbeli sorrendje):** Az `Array.prototype.forEach` szinkron módon azonnal lefut, nem várja meg az aszinkron (Promise) `fakePrice` hívások feloldódását. A kód rögtön beírja a 0 kezdőértéket a DOM-ba, miközben az árak aszinkron lekérdezése még csak később, a mikrotaszk sorból tér vissza.
- **Javítás (commit link):** https://github.com/AndreaJulia24/umfst-bh08-gyogyszerkereso/commit/b86cc4525355488ae6b6f531f8a7fe4632f4bb2f
- **Hogyan tesztelted:** Sikeres keresés után a felületen egy pici várakozás után megjelent a helyes, nullától eltérő összesített ár, mivel a kód most már egyszerre bevárta az összes ár lekérdezését.
- **Tanulság / érintett alapelv:** Az `Array.prototype.forEach` nem kezeli az aszinkron műveleteket. Párhuzamos hálózati kérések bevárásához a `Promise.all()` metódust kell használni.

## 5. hiba: Időzítő szivárgás (Timer Leak) az automatikus frissítésnél
- **Hiba leírása és reprodukálása:** Az „Automatikus frissítés” gombot többször megnyomva a lista villogni kezd, a hálózati kérések megsokszorozódnak (a CPU terhelés nő), és a folyamatot nem lehet kikapcsolni.
- **Gyökérok (eseményhurok, Promise-ok időbeli sorrendje):** Minden gombnyomás egy teljesen új `setInterval` időzítőt indít el. Mivel a kód nem menti el a visszatérő azonosítót, a korábbi időzítőket sosem törli. Így több kattintás után több időzítő fut párhuzamosan az eseményhurokban, végtelenül halmozva a kéréseket.
- **Javítás (commit link):** https://github.com/AndreaJulia24/umfst-bh08-gyogyszerkereso/commit/dd92a3fcc439f9557d36076185893e8777604b02
- **Hogyan tesztelted:** Bekapcsoltam az automatikus frissítést, majd rákattintottam újra. A gomb szövege megváltozott, az ismétlődő kérések pedig a konzolban láthatóan leálltak, a funkció sikeresen ki-be kapcsolható (toggle).
- **Tanulság / érintett alapelv:** Erőforrás-takarítás (Resource Cleanup). Az ismétlődő időzítőket (`setInterval`) mindig egy változóhoz kell rendelni, hogy később a `clearInterval` segítségével meg tudjuk állítani őket.

## Összegzés
## Mit tanultam:
A feladat során mélyebben megértettem az aszinkron JavaScript működését és az eseményhurok (event loop) sajátosságait. Megtanultam, hogyan védekezzek a versenyhelyzetek (race conditions) ellen, hogyan optimalizáljam a hálózati kéréseket debounce technikával, és miért elengedhetetlen a Promise-ok helyes bevárása (`Promise.all` a `forEach` helyett), valamint az aszinkron hibakezelés (`.catch()`). Ezen felül megtanultam figyelni az erőforrások tudatos kezelésére is (időzítők leállítása).
-
## Melyik alapelv sérült a leggyakrabban:
Egyértelműen az **aszinkron folyamatok (Promise-ok) és az eseményhurok helytelen kezelésének** elve sérült a legtöbbször. Az 1., 3. és 4. hibánál is abból fakadt a probléma, hogy az eredeti kód szinkron módon próbált kezelni aszinkron történéseket (nem vette figyelembe az időbeli csúszásokat, a mikrotask sor működését). Emellett az **erőforrás-menedzsment és optimalizáció** (debounce hiánya, lezáratlan `setInterval`) is visszatérő hiányosság volt.
