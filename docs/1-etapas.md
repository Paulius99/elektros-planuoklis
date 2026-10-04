# Elektros prietaisų įjungimo planuoklis

Kursinio darbo I dalis: projektavimo dokumentas

## 1. Problema ir idėja

**Sistema vienu sakiniu:** SaaS sistema namų ūkiams su dinaminiu elektros planu, kuri apskaičiuoja, kada įjungti prietaisus, kad elektra jiems kainuotų kuo mažiau.

**Problema ir dabartinis procesas:** vis daugiau žmonių renkasi planus, kur kaina susieta su Nord Pool birža (pvz. Ignitis „Išmanus 15“, Enefit „Laisvas“). Nuo 2025 m. spalio kaina keičiasi kas 15 min., tai per parą yra 96 kainos. Dabar žmogus pats turi kasdien tikrinti, kada elektra pigiausia. Net radęs pigų laiką, jis dažnai pamiršta tuo metu įjungti prietaisą, o kartais, užsivertęs kitais darbais, kainų išvis nepatikrina ir išleidžia daugiau, nei galėtų. Be to, pigiausias laikas priklauso nuo to, kiek prietaisas veikia, o pigiausia biržos kaina nebūtinai yra pigiausia galutinė, nes ESO persiuntimo kaina skiriasi pagal laiko zoną (4 zonų plane vakaras daugiau nei dvigubai brangesnis už naktį). Yra įrankių kaip Home Assistant ar EVCC, bet jiems reikia savo serverio, įrangos ir techninių žinių, be to, jie labiau skirti elektromobilių krovimui, o ne visiems namų prietaisams.

**Nauda:** prietaisus naudotojas įveda vieną kartą, o kasdien gauna planą su paleidimo laikais, kaina, sutaupymu ir paaiškinimu, kodėl pasirinktas toks laikas. Pagal planą jis iš karto nustato prietaisų atidėtą paleidimą, todėl nebereikia nei pačiam tikrinti kainų, nei prisiminti laiku įjungti prietaiso.

**Naudotojai:** namų ūkio savininkas nustato tarifą ir maržą, prideda prietaisus ir narius. Narys pateikia užklausą (pvz. „indaplovė turi baigti iki 7:00“) ir mato planą. Kiekvienas namų ūkis yra atskiras SaaS klientas (tenant) su savo duomenimis ir nustatymais. Toks modelis tiktų ir smulkiam verslui, pvz. skalbyklai.

**Prielaidos:** naudotojas turi išmanųjį skaitiklį ir dinaminį planą, tiekėjo maržą įveda pats, prietaisų galią žino iš etiketės. Sistema prietaisų nevaldo, tik pasako, kada juos įjungti.

## 2. Apimtis

| Funkcija | Ką naudotojas galės atlikti | Pagrindinis modulis ar pagalbinė funkcija |
|---|---|---|
| Namų ūkio paskyra ir rolės | Sukurti namų ūkį, pakviesti narius | Pagalbinė |
| Tarifo nustatymai | Pasirinkti ESO zonų planą (1, 2 ar 4 zonos), įvesti tiekėjo maržą | Pagalbinė |
| Prietaisų profiliai | Įvesti galią, trukmę ir ar programą galima pertraukti | Pagalbinė |
| Kainų gavimas | Sistema pati parsisiunčia kitos paros 15 min. kainas | Pagalbinė |
| Įjungimo plano sudarymas | Gauti paleidimo laikus, kainą, sutaupymą ir paaiškinimą | Pagrindinis modulis |

**Į kursinio darbo apimtį neįeina:** prietaisų valdymas (IoT), namo įvado galios ribos valdymas, kainų prognozavimas, mėnesio sąskaitos, mobili programėlė.

## 3. Pagrindinis modulis

**Pavadinimas ir atsakomybė:** Planuotojas. Iš paros kainų, tarifo ir prietaisų užklausų kiekvienam prietaisui parenka pigiausią laiką jo lange. Jei to padaryti negalima, grąžina aiškią klaidą.

**Logika, kurią reikės projektuoti ir testuoti:** galutinės kainos skaičiavimas kiekvienam intervalui pagal ESO zoną, pigiausio ištisinio lango paieška, pigiausių intervalų parinkimas pertraukiamiems prietaisams, lygių kainų atvejai, laiko keitimo paros.

**Įvestis:** paros biržos kainos, tarifas (zonų planas, marža, PVM) ir prietaisų užklausos. Pvz. indaplovė 2 kW, 30 min., nepertraukiama, langas 00:00-02:00.

**Išvestis:** kiekvienam prietaisui paleidimo intervalai, kaina, sutaupymas ir paaiškinimas. Pvz. „Indaplovė 00:30-01:00, 0,045 €, sutaupyta 0,045 € palyginus su paleidimu 00:00.“

**Veikimo eiga:** 1) patikrinami duomenys (ar yra visos lango kainos, ar trukmė telpa į langą); 2) kiekvienam intervalui apskaičiuojama galutinė kaina; 3) kiekvienam prietaisui pagal jo tipą parenkamas pigiausias blokas arba pigiausi intervalai; 4) grąžinamas planas su kainomis, sutaupymu ir paaiškinimais.

### Taisyklės arba sprendimo žingsniai

1. Trukmė apvalinama į didesnę pusę iki 15 min. intervalų (100 min. = 7 intervalai).
2. Intervalo kaina = (biržos kaina + marža) × (1 + PVM) + ESO zonos kaina. Zona nustatoma pagal vietos laiką, savaitės dieną ir vasaros ar žiemos laiką.
3. Nepertraukiamas prietaisas (indaplovė, skalbyklė) veikia vienu bloku, parenkamas blokas su mažiausia kainų suma.
4. Pertraukiamas prietaisas (elektromobilis, boileris) gauna reikiamą kiekį pigiausių intervalų lange, nebūtinai iš eilės.
5. Jei kainos vienodos, visada imamas ankstesnis laikas.
6. Jei lange trūksta bent vienos kainos arba trukmė ilgesnė už langą, prietaisui grąžinama klaida, o kiti prietaisai vis tiek suplanuojami.
7. Sutaupymas skaičiuojamas palyginus su paleidimu iškart lango pradžioje.
8. Neigiamos biržos kainos leidžiamos ir skaičiuojamos įprastai.

### Scenarijai būsimiems testams

| Scenarijus | Pradinės sąlygos ir konkreti įvestis | Veiksmas | Tikslus laukiamas rezultatas |
|---|---|---|---|
| Įprastas atvejis | Marža, PVM ir persiuntimas 0. Kainos 00:00-02:00 (€/kWh): 0,10; 0,08; 0,05; 0,04; 0,06; 0,09; 0,12; 0,11. Indaplovė 2 kW, 30 min., nepertraukiama, langas 00:00-02:00 | Sudaryti planą | Paleidimas 00:30, kaina 0,045 €, sutaupymas 0,045 € |
| Ribinis atvejis arba konfliktas | ESO zona pakeičia sprendimą. 2026-11-10 (antradienis), 4 zonų „Standartinis“ planas, PVM 21 %. Birža (€/kWh) 21:45 = 0,050, 22:00 = 0,060. Prietaisas 1 kW, 15 min., langas 21:45-22:15 | Sudaryti planą | 21:45 kaina 0,20691, 22:00 kaina 0,13552. Pasirenkama 22:00, nors birža brangesnė. Kaina 0,03388 € |
| Ribinis atvejis arba konfliktas | Lygios kainos. Marža, PVM ir persiuntimas 0. Kainos 02:00-04:00 (€/kWh): 0,05; 0,04; 0,03; 0,03; 0,04; 0,06; 0,07; 0,08. Boileris 3 kW, 45 min., pertraukiamas, langas 02:00-04:00 | Sudaryti planą | Intervalai 02:15, 02:30, 02:45 (iš dviejų 0,04 imamas ankstesnis). Kaina 0,075 €, sutaupymas 0,015 € |
| Klaida arba neįmanomas rezultatas | Elektromobilis 7,4 kW, 6 val., langas 01:00-05:00 | Sudaryti planą | Klaida „Trukmė viršija langą: reikia 24 intervalų, lange yra 16“ |

**Jei modulis naudoja AI:** Netaikoma.

## 4. Kokybės atributas

**Pasirinktas atributas:** modifikuojamumas.

**Kodėl svarbus šiai sistemai:** tarifai realiai keičiasi. Nord Pool 2025 m. perėjo prie 15 min. kainų, ESO turi 1, 2 ir 4 zonų planus ir keičia kainas, tiekėjai keičia savo planus. Jei kiekvienam pokyčiui reikėtų perrašyti planuotoją, sistema greitai taptų nepatikima.

**Tikrinimo scenarijus ir sąlygos:** prototipe bus 1 ir 2 zonų tarifai, vėliau pridėsiu 4 zonų.

**Sėkmės kriterijus:** pridedant 4 zonų tarifą sukuriama viena nauja klasė su testais, planuotojo failai nepakeisti, keičiama tik tarifų registracija (viena eilutė) ir konfigūracija. Visi ankstesni testai praeina.

**Numatytas projektavimo sprendimas:** tarifas skaičiuojamas per `TariffStrategy` sąsają (Strategy šablonas), kainos gaunamos per `PriceProvider` sąsają. Planuotojas nežino, koks tarifas ir iš kur kainos, ir neturi darbo su duomenų baze ar tinklu.

**Kaip patikrinsiu vėlesniame etape:** parodysiu pakeitimo `git diff` ir testų rezultatus prieš ir po.

**Sprendimo kaina arba ribojimas:** daugiau sąsajų ir klasių, iš pradžių kodas atrodys sudėtingesnis nei reikia.

## 5. Pradinė sistemos struktūra

### Paprasta schema

```
Naršyklė (web UI)
      |
      v
REST API (be būsenos, JWT su namų ūkio ID ir role)
      |
      +--> Planuotojas --> TariffStrategy (1 / 2 / 4 zonos)
      |
      +--> PriceProvider --> Elering API (atsarginis ENTSO-E, testams failai)
      |
      v
PostgreSQL (visos lentelės su household_id)
```

| Sistemos dalis | Atsakomybė |
|---|---|
| Web UI | Prietaisų ir tarifo įvedimas, plano peržiūra |
| REST API | Prisijungimas, rolės, klientų atskyrimas, užklausų tikrinimas |
| Planuotojas | Plano sudarymas pagal 3 skyriaus taisykles |
| TariffStrategy | Galutinės intervalo kainos skaičiavimas |
| PriceProvider | Kainų parsisiuntimas ir išsaugojimas |
| PostgreSQL | Namų ūkių duomenys, atskirti pagal `household_id` |

**Planuojamos technologijos ir pasirinkimo priežastys:** TypeScript ir Node.js (Express), nes JavaScript jau naudoju, o TypeScript sąsajos ir tipai tinka OOP ir SOLID. Testai su Jest, duomenų bazė PostgreSQL. Paleidimas per Docker Compose viena komanda, nustatymai per aplinkos kintamuosius. Kainos iš Elering API, nes jis nemokamas, be rakto ir grąžina LT zonos 15 min. kainas €/MWh (išbandžiau).

## 6. AI panaudojimas

### AI rengiant šį dokumentą

Taip, naudojau Claude (Anthropic).

| Priemonė ir užduotis | Ką panaudojau | Ką atmečiau arba perrašiau ir kodėl | Kaip patikrinau |
|---|---|---|---|
| Claude, idėjų paieška ir tikrinimas pagal reikalavimus | Iš 20 idėjų išsirinkau šią, gavau taisyklių ir scenarijų idėjas, pastabą dėl multitenancy | Atmečiau variantą su daugiabučio bendrija, AI pagrindiniame modulyje ir įvado galios ribos valdymą, kad apimtis būtų realesnė | Peržiūrėjau užduoties aprašą ir įvadinės paskaitos skaidres |
| Claude, duomenų prieinamumo patikra | Elering API, ESO 2026 m. tarifai, 15 min. kainų pradžia | Tarifų skaičius laikysiu konfigūracijoje, ne kode | API užklausos (gautos 92, 96 ir 100 intervalų paros), ESO svetainė |
| Claude, dokumento juodraštis | Struktūrą pagal šabloną ir formuluotes | Pataisiau netikslų teiginį apie EVCC, išėmiau antrą kokybės atributą ir kitą perteklinę informaciją | Scenarijų skaičius perskaičiavau ranka |

### Planuojamas AI naudojimas kuriant sistemą

**Kur ir kam naudosiu AI:** testų atvejams sugalvoti, kodo peržiūrai, refaktorinimo pasiūlymams, Docker konfigūracijai.

**Kaip tikrinsiu pasiūlymus ir sugeneruotą kodą:** pirma parašau testus pagal 3 skyriaus scenarijus, AI pasiūlytas kodas turi juos praeiti. Priimu tik tai, ką galiu paaiškinti. Kraštiniams atvejams (laiko keitimas, lygios ir neigiamos kainos) bus atskiri testai, nes ten AI dažniausiai suklysta.

**Ar AI bus sistemos funkcionalumo dalis:** Ne. Planas skaičiuojamas pagal aiškias taisykles, todėl rezultatą galima tiksliai patikrinti, o AI čia nieko nepagerintų.

## 7. Tolesnių darbų planas

| Darbas | Apčiuopiamas rezultatas | Planuojama darbų seka |
|---|---|---|
| Planuotojas ir 1, 2 zonų tarifai | 3 skyriaus scenarijai kaip unit testai, visi praeina | 1 |
| PriceProvider | Realios kainos iš Elering, testuose kainos iš failų | 2 |
| REST API ir multitenancy | Du namų ūkiai nemato vienas kito duomenų, veikia rolės | 3 |
| Web sąsaja, Docker Compose, README | Galima įvesti prietaisus ir pamatyti planą, paleidžiama viena komanda | 4 |

**Būsimo prototipo veikimo scenarijus:** įkeliu realias rytdienos LT kainas, pasirenku 2 zonų tarifą ir įvedu indaplovę, skalbyklę, elektromobilį ir boilerį su jų langais. Tikiuosi plano, kur kiekvienas prietaisas turi paleidimo laiką savo lange, matosi jo kaina, sutaupymas ir paaiškinimas.

| Rizika arba neaiškumas | Kaip patikrinsiu arba sumažinsiu |
|---|---|
| Elering API gali neveikti arba pakeisti formatą | `PriceProvider` sąsaja, atsarginis ENTSO-E, kainos failuose testams ir paleidimui be interneto |
| Laiko keitimo paros (92 ir 100 intervalų) | Viduje viskas UTC, testai su realiomis 2026-03-29 ir 2025-10-26 kainomis |
| Nord Pool para skaičiuojama pagal Vidurio Europos laiką, todėl LT paros pirmosios valandos kainos žinomos anksčiau nei likusios | Kainos saugomos pagal intervalą, o ne pagal parą, ir prieš planuojant tikrinama, ar lange yra visos kainos |

## Šaltiniai, jei naudojote

- ESO tarifų planai ir kainos 2026 m.: https://www.eso.lt/namams/elektra/tarifu-planai-kainos-atsiskaitymas/tarifu-planai-ir-kainos-2026-metais/4801
- Ignitis „Išmanus 15“: https://ignitis.lt/ismanus-15-elektros-tiekimo-planas
- Elering kainų API: https://dashboard.elering.ee/api/nps/price
- ENTSO-E Transparency Platform: https://transparency.entsoe.eu
