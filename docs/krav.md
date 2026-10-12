# Krav — informationsaccess

Krav på det event som publiceras när information har visats för en användare.
Eventet definieras i `rimfrost-service-informationsaccess-asyncapi` och publiceras av
BFF:erna.

**Underlag:** FKPOC-1080 (ILOGG integration), som i sin tur hänvisar till FK:s FIFF-181 och
DST-425. FIFF-181 och DST-425 är inte lästa — kraven nedan är härledda ur beskrivningen i
FKPOC-1080 och ur diskussioner i teamet.

**Omfattning:** kraven gäller eventkontraktet i det här repot, dvs. **FKPOC-1097**, vars spec
är mergad i PR #1 och vars ärende står som Klart. Dokumentet är skrivet i efterhand för att
göra de antaganden som specen vilar på granskningsbara.

`visadInformation` sköts i PR #1 medvetet upp till **FKPOC-1103**, med motiveringen att ett
tillagt valfritt fält inte bryter befintliga konsumenter. Kraven på det fältet (IA-FR-04)
finns därför med här, men är inte infriade i dagens kontrakt.

Alla ärenden ligger under **FKPOC-1101** (Logga uppgiftsaccess via nytt kafka interface), som
i sin tur ligger under **FKPOC-1080**. Varken 1101 eller 1097 har någon beskrivning — 1080 är
den enda källan med innehåll.

**Status:** utkast.

## Märkning av krav

| Märkning | Betydelse |
|---|---|
| **[1080]** | Följer av beskrivningen i FKPOC-1080. |
| **[1097]** | Redan beslutat i PR #1 (Lisa Wedin, godkänd av Ulf Slunga 2026-10-01) och infört i `asyncapi.yaml`. |
| **[förslag]** | Förslag från Rimfrost. Inte förankrat med FK — behöver bekräftas. |

En stor del av kraven är **[förslag]**. De är framtagna genom tolkning av diskussioner inom
teamet, inte ur ett beslutat underlag, och ska läsas som utgångspunkt för ett samtal — inte
som fastställda krav.

Frågor som ännu inte är besvarade står inte som krav, utan samlat under **Öppna punkter**.

## Arkitektur

**[1080]** Rimfrost publicerar ett datapaket på en Kafkatopic. **Mimer** konsumerar topicen,
knyter uppgiftens ID till ett processID (handläggning) med hjälp av ärendedata Mimer redan
har, och ansvarar för att rätt datapost skickas vidare till den riktiga ILOGG:en. Eftersom
både uppgiften och processen har specifikationer går det att utläsa i vilket sammanhang och
med vilket syfte handläggaren tagit del av personuppgifterna.

Rimfrosts ansvar slutar vid det publicerade eventet.

## Avgränsning

Utanför kraven ligger, med ärende och ansvarig enligt FKPOC-1101:

- **FKPOC-1098** (Lars Persson) — BFF:en använder asyncapi:t för att logga uppgiftsaccess.
  Producentens implementation.
- **FKPOC-1103** (Julia Olsson) — utred vilken extra information som behöver tillföras till
  reglers get response, så att BFF får det som krävs för att fylla eventet.
- **FKPOC-1102** (Julia Olsson) — scenario med uppgiftsaccess-event i `rimfrost-kubernetes`.
- **FKPOC-1099** (Klart) — borttagen befintlig loggning av uppgiftsaccess till förmån för
  async i BFF.

Utanför ligger också:

- Mimers koppling uppgift → process och Mimers leverans till ILOGG
- hur ett registerutdrag till en enskild individ faktiskt produceras — det sker i ILOGG
- behörighetskontroll, som ligger kvar hos de bakomliggande tjänsterna

## Funktionella krav

### IA-FR-01 — När eventet ska publiceras

- **IA-FR-01.1** *[1097]* Ett event ska publiceras varje gång information om en eller flera
  individer lämnar en BFF till klienten. Ursprungligen omfattade specen bara att en
  handläggare öppnar en regel; i PR #1 breddades den medvetet till att gälla alla användare
  ("model as event for all users"). FKPOC-1080 beskriver fortfarande bara det som visas i en
  manuell uppgifts GUI, så breddningen är tagen på Rimfrosts eget initiativ och är inte
  förankrad hos FK. Kravet är i praktiken snävare än det står: vyer utan uppgift är parkerade
  tills vidare (se Öppna punkter), och de två kan inte båda gälla fullt ut förrän den frågan
  är besvarad.
- **IA-FR-01.2** *[förslag]* Avgörande är vad som går från BFF till klient. Information som
  hämtats internt i backend men aldrig skickats vidare ska inte loggas, och det ska inte
  heller krävas kännedom om vad klienten faktiskt renderat på skärmen.
- **IA-FR-01.3** *[förslag]* Om ett svar innehåller flera uppgifter, t.ex. en uppgiftslista,
  publiceras ett event per uppgift — inte ett för hela svaret. Mimer kopplar via uppgiftens
  ID, och IA-FR-02.2 kräver ändå ett event per individ.
- **IA-FR-01.4** *[förslag]* Flera BFF:er kan publicera var sitt event för det användaren
  upplever som en sammanhängande handling, t.ex. en uppgiftslista i portalen följd av att
  en uppgift öppnas i en regel. Det är avsiktligt och ska inte behandlas som dubbletter.

### IA-FR-02 — Separerbarhet per individ

Enligt FKPOC-1080 sker uppdelningen per individ hos Mimer, genom kopplingen uppgift →
process → ärendedata. Den kopplingen bär bara om datapaketet rör exakt en individ.

> **Antagande, delvis verifierat 2026-10-12:** en regels svar rör en individ i taget.
> Kontrollerat mot `rimfrost-regel-rtf-manuell-bff` och `rimfrost-regel-bekraftabeslut-bff` —
> båda returnerar `kund` i singular, utan lista. Kraven nedan vilar på det.
>
> Det gäller **inte** portalens uppgiftslista. `OperativUppgift` i `rimfrost-portal-bff` bär
> `individer` som lista, och OUL:s asyncapi beskriver fältet som "Lista av individer i
> ärendet". Den vyn är parkerad, se Öppna punkter. Tas den in i omfattningen behöver
> `individer` bära visad information per individ, vilket är en brytande kontraktsändring.

- **IA-FR-02.1** *[förslag]* Loggen ska gå att dela upp per individ, så att ett utdrag till
  en enskild person kan produceras utan att röja uppgifter om någon annan individ.
- **IA-FR-02.2** *[förslag]* Ett event ska röra exakt en individ. Så länge det gäller är
  eventnivån också individnivån, och `visadInformation` kan ligga kvar på eventnivå utan att
  separerbarheten i IA-FR-02.1 går förlorad.
- **IA-FR-02.3** *[förslag]* Om ett svar någon gång visar uppgifter om flera individer ska
  producenten publicera ett event per individ, inte ett event med flera. Det är en regel för
  producenten och kräver ingen ändring av kontraktet.

### IA-FR-03 — Vilka individer som ingår

- **IA-FR-03.1** *[1097]* Eventet ska lista de individer vars information faktiskt
  passerade BFF:en till klienten i just det svaret — inte samtliga individer som är knutna
  till ärendet.
- **IA-FR-03.2** *[förslag]* Identifieraren för en individ finns i eventet för loggens
  skull. Att en individ förekommer i eventet ska inte tolkas som att identifieraren i sig
  visades för användaren.

### IA-FR-04 — Visad information

Fältet `visadInformation` utelämnades medvetet i PR #1 och hör till **FKPOC-1103**.
Motiveringen var att ett tillagt valfritt fält inte bryter befintliga konsumenter, så
beslutet kunde skjutas upp. Kraven nedan är alltså inte infriade i dagens kontrakt.

- **IA-FR-04.1** *[1080]* Eventet ska innehålla den data som hämtades för att visas för
  användaren. FKPOC-1080: *"datapaketet innehåller det data som en användare hämtar för att
  visa i en manuell uppgifts GUI."*
- **IA-FR-04.2** *[förslag]* Datan ligger på eventnivå. Eftersom ett event rör exakt en
  individ (IA-FR-02.2) är det detsamma som per individ.

### IA-FR-05 — Användare, tidpunkt och källa

- **IA-FR-05.1** *[1080]* Eventet ska ange vilken användare som fick informationen visad.
  FKPOC-1080 anger kortnummer. PR #1 noterar att id-värdet i exemplet är påhittat och kan
  behöva bytas mot det riktiga formatet — den kontrollen är fortfarande ogjord.
- **IA-FR-05.2** *[1080]* Eventet ska ange när informationen visades.
- **IA-FR-05.3** *[1097]* Eventet ska ange vilken applikation som visade informationen och
  publicerade eventet.
- **IA-FR-05.4** *[1097]* Användaren behöver inte vara en handläggare. Om en person tar del
  av sina egna uppgifter i en självbetjäningstjänst är användaren samma person som den
  individ informationen rör, och eventet ska publiceras på samma sätt. Det följer av
  breddningen i PR #1 (se IA-FR-01.1). FKPOC-1080 beskriver endast handläggarfallet.

### IA-FR-06 — Sammanhang

- **IA-FR-06.1** *[1080]* Eventet ska ange vilken uppgift informationen visades i. Mimers
  koppling till handläggningen bygger på uppgiftens ID, vilket gör fältet nödvändigt och
  inte valfritt. `uppgiftId` gjordes obligatoriskt i PR #1 (commit 32e3499) men blev valfritt
  igen när specen breddades till alla användare (commit 1ab0786). Konflikten är inte löst.
- **IA-FR-06.2** *[1097]* Eventet ska, när det är tillämpligt, ange vilket ärende och vilken
  regel informationen visades i. Regeln identifieras med `regelId` och `regelVersion` från
  regelns `config.yaml`, inte med fritextnamn, så att loggarna går att söka entydigt.
- **IA-FR-06.3** *[förslag]* Eventet bör gå att koppla till det anrop som utlöste det, så
  att en loggpost kan härledas till en specifik begäran vid felsökning och granskning.

### IA-FR-07 — Tillförlitlighet vid publicering

- **IA-FR-07.1** *[förslag]* Om ett misslyckat publiceringsförsök inte stoppar utlämnandet
  ska det framgå i den publicerande applikationens egen loggning, så att tappade händelser
  går att upptäcka i efterhand.

### IA-FR-08 — Identitet och dubbletter

- **IA-FR-08.1** *[förslag]* Varje event ska ha en egen identifierare, så att konsumenter
  kan skilja två separata visningar från samma visning levererad mer än en gång.
- **IA-FR-08.2** *[förslag]* Konsumenter ska kunna behandla eventströmmen idempotent.
  Eftersom leveransen är at-least-once kan samma event komma flera gånger, och det får inte
  leda till dubbla poster i ett registerutdrag.

### IA-FR-09 — Topic

- **IA-FR-09.1** *[1080]* Alla event publiceras på en och samma topic. Inga subtopics, t.ex.
  per handläggningsid.
- **IA-FR-09.2** *[1080]* Topicen konfigureras statiskt, inte via dynamisk lookup.

## Icke-funktionella krav

### IA-NFR-01 — Personuppgifter

- **IA-NFR-01.1** *[1080]* Eventet innehåller personuppgifter — både individernas
  identifierare och den data som visades (IA-FR-04.1) — och ska behandlas därefter i
  lagring, åtkomst och gallring.

### IA-NFR-02 — Utvecklingsbarhet

- **IA-NFR-02.1** *[förslag]* Kontraktet ska kunna utökas med mer information per individ
  utan brytande ändringar för befintliga producenter och konsumenter.

### IA-NFR-03 — Testbarhet

- **IA-NFR-03.1** *[förslag]* Att eventet publiceras med rätt innehåll ska gå att verifiera
  i integrationstest, utan att behöva läsa producentens interna tillstånd.

## Konsekvenser för nuvarande kontrakt

Kontraktet ligger i dag på 0.1.0 och har inga implementationer. Ändringarna nedan följer av
kraven, men bör inte göras förrän de öppna punkterna är besvarade.

Ingen av dem är brytande, så länge antagandet i IA-FR-02 håller. Håller det inte behöver
`individer` göras om till objekt som bär visad information per individ, vilket *är* en
brytande ändring.

1. `visadInformation` är uppskjutet till FKPOC-1103 enligt beslutet i PR #1. Enligt
   IA-FR-04.1 ska det bära den faktiska datan, och enligt IA-FR-04.2 ligger det på eventnivå.
   Strukturen är inte beslutad, se Öppna punkter. Att lägga till fältet som valfritt bryter
   inte befintliga
   konsumenter.
2. Eventet saknar en egen identifierare. IA-FR-08.1 kräver ett sådant fält.
3. `uppgiftId` är i dag valfritt. IA-FR-06.1 gör det nödvändigt. Beslutet hänger ihop med
   frågan om vyer utan uppgift ska ingå — i så fall behöver Mimer något annat att koppla på.
4. `individer` har i dag `minItems: 1` utan övre gräns. Enligt IA-FR-02.2 rör ett event exakt
   en individ. Antingen sätts `maxItems: 1`, eller så lämnas listan som den är och regeln
   upprätthålls av producenten. Det senare är mer förlåtande om antagandet visar sig fel.
5. `anvandare` behöver bekräftas bära kortnummer enligt IA-FR-05.1. Verifieras mot
   `anropare` i FKPOC-1098 och FKPOC-1103.

## Öppna punkter

Kolumnen *Beslutas av* är bara ifylld där det finns belägg för det. Där den säger
"ej utrett" behöver frågan om ägarskap ställas innan frågan i sig kan drivas — börja med
Lars eller Ulf. Lisa skrev den ursprungliga specen och är närmast kontraktsfrågorna.

| Punkt | Berör | Beslutas av |
|---|---|---|
| Vyer utan uppgift, t.ex. OUL-listor | IA-FR-06 | Jonas (enligt Lars) — avgör även IA-FR-02 |
| Struktur och granularitet på den visade datan | IA-FR-04 | Jonas |
| Om en BFF-respons kan visa uppgifter om flera individer | IA-FR-02 | Besvarat för regler: nej. Öppet för uppgiftslistan, som är parkerad |
| Om information som inte är personbunden ska loggas | IA-FR-02 | Ej utrett |
| Om datan får reduceras till det som gick till klienten | IA-FR-04 | Ej utrett |
| Om utlämnande ska stoppas när eventet inte kan publiceras | IA-FR-07 | Ej utrett |
| Om kortnummer finns i Rimfrosts anroparkontext | IA-FR-05 | Rimfrost, i FKPOC-1098/1103 |
| Vilka `typId`-UUID:n som gäller för kortnummer och personnummer | IA-FR-03 | Rimfrost — Lisa närmast |
| Om självåtkomst alls ska loggas | IA-FR-05 | Ej utrett — FK-fråga |
| Vad eventen partitioneras på | IA-FR-09 | Ej utrett |
| Övre gräns för meddelandestorlek när datan ingår i eventet | IA-FR-04 | Ej utrett |
