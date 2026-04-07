# MVT GUI Wrapper - Technické zhrnutie

## 1. Cieľ projektu a rozsah

Cieľom riešenia bolo vytvoriť použiteľný prvý desktop wrapper nad nástrojom Mobile Verification Toolkit (MVT), ktorý bude fungovať na Windows a umožní bežnému používateľovi pracovať s MVT bez manuálneho písania príkazov v termináli. Zadanie kládlo dôraz na štyri hlavné priority: jednoduchosť návrhu, rýchlu implementáciu, jednoduché spúšťanie na Windows a ľahkú údržbu.

Prakticky to znamenalo navrhnúť GUI, ktoré:
- umožní zvoliť platformu (iOS/Android),
- sprístupní základné workflow relevantné pre MVT,
- poskytne pohodlný výber vstupov a výstupného priečinka,
- spustí analýzu na pozadí bez zablokovania používateľského rozhrania,
- transparentne ukáže, čo sa deje počas behu (príkaz, log, stav, výsledok),
- po dokončení uľahčí prácu s výsledkami.

Výsledná implementácia pokrýva:
- výber platformy (`iOS`, `Android`),
- výber workflow (`backup`, `filesystem`, `adb` podľa platformy),
- výber vstupných priečinkov/súborov a IOC súboru,
- voľbu output priečinka,
- spúšťanie MVT v background režime s live logom,
- indikáciu priebehu a stavu úlohy,
- sumár po dokončení vrátane otvorenia priečinka s výsledkami,
- parsovaný report z JSON výstupov v tabuľke,
- export reportu do formátov CSV a HTML,
- bezpečné prerušenie bežiacej úlohy tlačidlom Stop.

## 2. Zvolený prístup a architektúra

Architektúra je zámerne jednoduchá a pragmatická:
- jazyk: Python,
- GUI vrstva: Tkinter (štandardná súčasť Pythonu),
- jadro analýzy: knižnica `mvt`,
- komunikácia medzi vláknami/procesmi: `queue` + periodické spracovanie udalostí v GUI loope.

### Prečo tento stack
Vybraný stack najlepšie zodpovedá cieľu „funkčný prvý wrapper“:
- netreba zavádzať komplexný frontend framework,
- deployment na Windows je priamočiary (`python app.py`),
- kód je čitateľný a ľahko rozšíriteľný,
- odstránili sa zbytočné vrstvy, ktoré by spomaľovali vývoj.

### Kľúčové technické rozhodnutia
1. **Jedna hlavná aplikácia s jasnými sekciami**
   Konfigurácia runu, runtime panel, log panel, report panel a export sú implementované v jednom module. Pri tomto rozsahu je to prehľadnejšie než zložitá viacvrstvová architektúra.

2. **Mapovanie GUI voľby na validné MVT commandy**
   Workflow v GUI nie sú len textové štítky. Aplikácia ich mapuje na konkrétne príkazy MVT podľa platformy (napr. iOS filesystem → `check-fs`, Android filesystem-like akvizícia → `check-androidqf`).

3. **Nezablokovanie GUI**
   Orchestrácia behu je vo worker vlákne, pričom samotné spustenie MVT beží v samostatnom procese. Toto rieši stabilitu UI aj možnosť bezpečného stopnutia.

4. **Bezpečný tok udalostí**
   Všetky runtime udalosti (status, log, progress, done) idú cez frontu. GUI sa aktualizuje iba z hlavného vlákna, čím sa predchádza race conditions a náhodným chybám v Tkinteri.

## 3. Komunikácia GUI s MVT

Komunikácia medzi GUI a MVT má jasný životný cyklus:

1. **Vstupná konfigurácia**
   Používateľ vyberie platformu, workflow, vstupy, IOC a output priečinok. Pred spustením sa overia povinné polia a existencia zadaných ciest (s výnimkami podľa workflow, napr. ADB).

2. **Zostavenie command modelu**
   GUI pripraví interný objekt s:
   - cieľovou platformou,
   - argumentmi pre MVT,
   - „display“ verziou príkazu pre UX.

3. **Spustenie v samostatnom procese**
   Worker vlákno vytvorí child proces, ktorý zavolá MVT entrypoint (`mvt-ios` / `mvt-android`) programovo, s konkrétnymi argumentmi.

4. **Zber logov a stavu**
   `stdout`/`stderr` sa presmerujú do queue writeru. GUI pravidelne odčítava frontu a zobrazuje:
   - aktuálny status,
   - bežiaci command,
   - live log.

5. **Progress a finalizácia**
   Progress je odhadovaný podľa dostupných percent v logu (best-effort). Po dokončení sa nastaví výsledný stav (success/error/cancelled), vygeneruje sumár, odomkne otvorenie output priečinka a spustí sa načítanie reportu.

6. **Stop/Cancellation**
   Pri stlačení Stop sa nastaví príznak zrušenia. Ak proces stále beží, je ukončený, čo umožňuje prakticky okamžitú reakciu bez zamrznutia aplikácie.

## 4. Parsovanie reportu a export

Po dokončení runu aplikácia prejde JSON výstupy v output priečinku a zobrazí ich v „Parsed Report“ tabuľke. Keďže rôzne MVT moduly vracajú rôzne štruktúry, parser je navrhnutý schema-agnosticky:
- ak nájde zoznamy, rozbalí ich do riadkov,
- ak nájde slovníky, hľadá najpravdepodobnejšie polia (`severity`, `module`, `indicator`, `value` a podobné),
- pri neznámej štruktúre stále zobrazí aspoň reprezentatívny text.

Tabuľka reportu obsahuje:
- zdrojový JSON súbor,
- index položky,
- odhad severity,
- modul/zdroj,
- indikátor,
- hodnotu (skrácenú pre čitateľnosť).

### Export reportu
Implementovaný export:
- **CSV**: vhodné pre ďalšiu analýzu v Exceli alebo LibreOffice,
- **HTML**: vhodné pre zdieľanie, tlač alebo jednoduchý manažérsky prehľad.

Export používa aktuálne načítané riadky z GUI tabuľky, teda používateľ exportuje presne to, čo vidí.

## 5. Čo bolo komplikované

Najnáročnejšie časti riešenia boli tieto:

1. **Responzívnosť GUI pri dlhých úlohách**
   MVT analýza môže trvať výrazne dlhšie. Pri priamom behu v GUI vlákne by aplikácia zamrzla. Riešením bolo oddeliť vykonávanie do worker vrstvy.

2. **Spoľahlivý Stop mechanizmus**
   Pri bezpečnom prerušení bolo potrebné vyhnúť sa „visiacim“ taskom. Preto bol zvolený child proces model, kde je možné aktívny príkaz ukončiť bez poškodenia GUI stavu.

3. **Logy s ANSI escape sekvenciami**
   Výstup MVT (a knižníc okolo neho) môže obsahovať ANSI štýly. Bolo nutné riešiť ich interpretáciu, aby log nebol nečitateľný.

4. **Klikateľné URL v logu**
   Priamo v logu sa objavujú odkazy. Implementácia rieši detekciu URL, kliknutie a bezpečnostné potvrdenie pred otvorením.

5. **Heterogénne JSON schémy**
   MVT moduly negarantujú jednotný „report schema“. Preto parser funguje heuristicky a „best-effort“, aby bol prakticky použiteľný aj pri rozdielnych moduloch.

## 6. Aktuálne obmedzenia

Aktuálna verzia je funkčná, ale má vedomé limity:

- Nie sú vystavené všetky pokročilé MVT parametre v GUI (module-level prepínače, špecifické voľby podľa commandu).
- Progress je odvodený z logu (best-effort), nie z oficiálneho progress API.
- Parsovaný report je heuristický, nie striktne viazaný na jeden pevný JSON model.
- ADB workflow je závislý od externého prostredia (zariadenie, debug bridge, oprávnenia).
- Distribúcia ako `.exe` zatiaľ nie je súčasťou odovzdaného scope.

## 7. Ďalšie kroky

Odporúčané pokračovanie vývoja:

1. **Rozšírenie GUI o advanced voľby**
   Doplniť per-workflow konfiguráciu (verbose mód, modulové filtre, ďalšie argumenty).

2. **Vylepšenie reportingu**
   Pridať filtrovanie, fulltext vyhľadávanie, triedenie stĺpcov a prípadný PDF export.

3. **Automatizované testy**
   Doplniť smoke testy pre mapovanie commandov a jednotkové testy pre parser reportu.

4. **Packaging pre odovzdanie**
   Pripraviť `.exe` build cez PyInstaller a krátky release postup pre Windows.

## 8. Záver

Výsledkom je funkčný, praktický a ľahko pochopiteľný desktop wrapper nad MVT pre Windows. Aplikácia pokrýva požadované minimum a zároveň obsahuje užitočné nadstavby (stop runu, parsovaný report, export CSV/HTML), ktoré zvyšujú použiteľnosť pri reálnej práci.

Z pohľadu zadania bol zachovaný správny kompromis medzi jednoduchosťou a funkčnosťou: riešenie nie je prekomplikované, je ľahko spustiteľné, má zrozumiteľný kód a poskytuje dobrý základ pre ďalšie iterácie.
