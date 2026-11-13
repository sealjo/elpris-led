# Förslag: dagen plus två lampor för i morgon

Ett förslag på en annan färglogik för Elpris-LED, tänkt för den som snabbt vill se om det lönar sig att flytta tvätt, disk eller bad. Det går att prova i `index.html` i den här mappen. Den första emulatorn i `docs/emulator/` visar originalet och är oförändrad.

## Tanken

Originalet färgar varje dag från billigast (grönt) till dyrast (rött) inom det som visas. En dag där priset bara rör sig mellan 30 och 40 öre blir då lika röd och grön som en dag mellan 20 och 300 öre, fast det i praktiken inte spelar någon roll när man tvättar. Förslaget jämför i stället med ett normalpris och har en neutral zon för små skillnader. Det svarar dessutom på frågan "är i morgon dyrare eller billigare än i dag?" med två egna lampor.

## Upplägg på slingan

Med `N` lampor:

| Lampor | Visar |
|---|---|
| `0` till `N-4` | I dag från lokal midnatt till midnatt, som i originalet men på `N-3` lampor |
| `N-3` | Alltid släckt, skiljer dagen från morgondagen |
| `N-2` | I morgon dag, 08–17, jämfört med i dag 08–17 |
| `N-1` | I morgon kväll, 17–24, jämfört med i dag 17–24 |

Dagens del använder samma `ledSlotRange` som i dag, med `numLeds = N-3` och dagens antal kvartar (96, eller 92/100 vid omställning till och från sommartid). Varje lampa får medelpriset av sina kvartar. Morgondagen ritas inte ut på slingan, den sammanfattas bara i de två sista lamporna.

Det som finns kvar från originalet gäller hela slingan: nu-markeringen pulserar mot vitt, passerad tid dämpas, nattljusstyrkan används och omvänd riktning (`physicalLed`) vänder hela slingan, inklusive de tre sista lamporna. De två morgonlamporna räknas alltid som framtid, så de dämpas inte och pulserar inte.

## Normalpris

Normalpriset är medelvärdet av alla kvartspriser för:

- de `H` senaste hela dygnen (standard 3, inställbart 0–7),
- i dag,
- i morgon, när morgondagens priser finns.

Att morgondagen räknas in gör att dagens färger kan ändras när morgondagens priser hämtas efter lunch. En mycket dyr morgondag höjer normalpriset, och då blir i dag grönare, vilket stämmer med rådet "tvätta i dag". Vill man att dagens färger står still under dagen kan morgondagen uteslutas ur normalpriset.

Historiken hämtas från samma adress som i dag med tidigare datum, ett anrop per dygn. Det behövs bara en summa och ett antal per dygn, inte varje kvart. ESP8266 har bara plats för 256 kvartar i cachen, alltså ungefär 2,5 dygn, så historiken bör sparas som dygnsmedel och inte i ringbufferten.

## Lägen och färger

För dagens lampor: `d = pris - normalpris`. För morgonlamporna: `d = snitt i morgon - snitt i dag` för samma tidsperiod. Samma gränser används för båda:

| Läge | Villkor (standard) | Hue (FastLED, `CHSV(hue, 255, 255)`) | Färg |
|---|---|---|---|
| Mycket billigt / mycket billigare | `d <= -80 öre`, eller pris `<= 0` (bara dagens lampor) | 96 | grönt |
| Billigt / billigare | `d <= -20 öre` | 80 | gulgrönt |
| Spelar ingen roll / ungefär som i dag | `-20 < d < 20 öre` | 64 | gult |
| Dyrt / dyrare | `d >= 20 öre` | 32 | orange |
| Mycket dyrt / mycket dyrare | `d >= 80 öre` | 0 | rött |

Gränserna på 20 och 80 öre gäller spotpris utan moms, som allt annat i lampan. Det finns också ett läge med bara tre nivåer (grönt 96, gult 64, rött 0), där "mycket" slås ihop med "billigt" och "dyrt". Gulgrönt och grönt kan vara svåra att skilja på en riktig slinga, så tre nivåer kan vara tydligare.

Ingenting blinkar eller andas, utom nu-markeringen som i dag.

## Morgonlamporna

- **Perioder:** 08–17 (kvart 32–67) och 17–24 (kvart 68–95) i lokal tid. Vid omställning till och från sommartid bör perioderna räknas från lokal tid, inte från kvartsnummer.
- **Innan morgondagens priser finns** (samma kontroll som `dayCovered` för morgondagen) är båda släckta.
- **Snitt:** både i dag och i morgon är snittet för hela perioden, även den del som redan har passerat i dag.

## Möjliga inställningar i webbgränssnittet

| Inställning | Standard |
|---|---|
| Normalpris, dygn bakåt | 3 |
| Spelar ingen roll inom | ±20 öre |
| Mycket dyrt eller billigt från | ±80 öre |
| Antal lägen | 5 (eller 3) |
| Räkna in morgondagen i normalpriset | på |

## Förenklingar i emulatorn

Emulatorn räknar med 96 kvartar per dygn och bortser från dygn med 23 eller 25 timmar. Exempeldagarna och de sju dygnen bakåt är påhittade. Inklistrade riktiga priser har ingen historik, och då bygger normalpriset bara på i dag och i morgon. Klockan startar på 14:00 så att morgonlamporna syns direkt. Lampan själv börjar leta efter morgondagens priser 13:30.
