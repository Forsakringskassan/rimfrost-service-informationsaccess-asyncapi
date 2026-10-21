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
tillagt valfritt fält inte bryter befintliga konsumenter. Fältet har nu lagts till i
kontraktets version 0.2.0.

Alla ärenden ligger under **FKPOC-1101** (Logga uppgiftsaccess via nytt kafka interface), som
i sin tur ligger under **FKPOC-1080**. Varken 1101 eller 1097 har någon beskrivning — 1080 är
den enda källan med innehåll.

**Status:** utkast.

## Märkning av krav

| Märkning | Betydelse |
|---|---|
| **[1080]** | Följer av beskrivningen i FKPOC-1080. |
| **[1097]** | Redan beslutat i PR #1 och infört i `asyncapi.yaml`. |
| **[förslag]** | Förslag från Rimfrost. Inte förankrat med FK — behöver bekräftas. |
| **[FK-svar]** | Besvarat av FK. |

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

Utanför kraven ligger, med ärende enligt FKPOC-1101:

- **FKPOC-1098** — BFF:en använder asyncapi:t för att logga uppgiftsaccess.
  Producentens implementation.
- **FKPOC-1103** — utred vilken extra information som behöver tillföras till
  reglers get response, så att BFF får det som krävs för att fylla eventet.
- **FKPOC-1102** — scenario med uppgiftsaccess-event i `rimfrost-kubernetes`.
- **FKPOC-1099** (Klart) — borttagen befintlig loggning av uppgiftsaccess till förmån för
  async i BFF.

Utanför ligger också:

- Mimers koppling uppgift → process och Mimers leverans till ILOGG
- hur ett registerutdrag till en enskild individ faktiskt produceras — det sker i ILOGG
- behörighetskontroll, som ligger kvar hos de bakomliggande tjänsterna

## Funktionella krav

### IA-FR-01 — När eventet ska publiceras

- **IA-FR-01.1** *[FK-svar]* Ett event ska publiceras varje gång **personuppgifter** om en
  eller flera individer lämnar en BFF till klienten. Utlösaren är personuppgifterna, inte vyn
  och inte uppgiften: en vy som inte bär personuppgifter ska inte loggas.
  I PR #1 breddades specen medvetet från handläggare till alla användare ("model as event for
  all users"). Den breddningen är fortfarande Rimfrosts eget initiativ och inte förankrad hos
  FK — se IA-FR-05.4.
- **IA-FR-01.2** *[FK-svar]* Avgörande är vad som går från BFF till klient. Information som
  hämtats internt i backend men aldrig skickats vidare ska inte loggas, och det ska inte
  heller krävas kännedom om vad klienten faktiskt renderat på skärmen. Loggningen sker när
  datan lämnar BFF:en, varje gång GUI:t laddas. Varje laddning loggas för sig, även när samma
  data visas igen.
- **IA-FR-01.3** *[förslag]* Det användaren upplever som en sammanhängande handling kan ge
  flera event, publicerade från olika steg och av olika anledningar. Det är avsiktligt och
  ska inte behandlas som dubbletter.

> **Besvarat av FK:** det tidigare antagandet, att loggningen utlöses när en uppgift öppnas
> och inte när en uppgiftslista visas, är överspelat. Loggningen sker vid varje GUI-laddning,
> och en uppgiftslista undantas inte som vy — den undantas bara om den saknar personuppgifter.
> FK:s inriktning är att hålla nere uppgiftsinformationen i OUL, så att listan inte bär
> personuppgifter och inte behöver loggas. Portalens uppgiftslista bär i dag personuppgifter,
> se Öppna punkter.

### IA-FR-02 — Ett event per visning

Enligt FKPOC-1080 sker uppdelningen per individ hos Mimer, genom kopplingen uppgift →
process → ärendedata. Det är alltså inget Rimfrost behöver lösa i kontraktet, och ett
registerutdrag per individ kan produceras utan att eventet delas upp.

- **IA-FR-02.1** *[FK-svar]* Allt som visades vid ett tillfälle ska ligga i samma meddelande.
  Eventet ska inte delas upp i flera, varken per individ eller per uppgift. FKPOC-1080:
  *"ett datapaket på en Kafkatopic som Mimer konsumerar."* Datapaketet är det eller de
  JSON-objekt som visas i GUI:t vid tillfället.

> **Rationale:** ett tidigare utkast krävde ett event per individ. Det är struket. En
> uppdelning gör den totala bilden av vad som visades vid en viss tidpunkt svårare att
> sammanställa, en konsument kan inte veta att den fått alla delar utan att eventet bär ett
> förväntat antal, och det finns ingen väg att be om meddelande 5 av 7 igen.

> **Kontrollerat:** ett regelsvar rör en individ i taget.
> `rimfrost-regel-rtf-manuell-bff` och `rimfrost-regel-bekraftabeslut-bff` returnerar båda
> `kund` i singular, utan lista. Det gäller **inte** portalens uppgiftslista —
> `OperativUppgift` i `rimfrost-portal-bff` bär `individer` som lista, alltså personuppgifter.
> Enligt IA-FR-01.1 ska den vyn därmed loggas, om den inte först bantas. Se Öppna punkter.

### IA-FR-03 — Vilka individer som ingår

- **IA-FR-03.1** *[1097]* Om eventet listar individer ska det vara de individer vars
  information faktiskt passerade BFF:en till klienten i just det svaret — inte samtliga
  individer som är knutna till ärendet.
- **IA-FR-03.2** *[förslag]* Identifieraren för en individ finns i eventet för loggens
  skull. Att en individ förekommer i eventet ska inte tolkas som att identifieraren i sig
  visades för användaren.
- **IA-FR-03.3** *[FK-svar]* Individerna behöver inte listas separat. Personuppgifterna finns
  i den visade datan (IA-FR-04.1), och det räcker för FK.

### IA-FR-04 — Visad information

Fältet `visadInformation` utelämnades medvetet i PR #1 och hör till **FKPOC-1103**.
Motiveringen var att ett tillagt valfritt fält inte bryter befintliga konsumenter, så
beslutet kunde skjutas upp. Fältet finns i kontraktet från version 0.2.0.

- **IA-FR-04.1** *[FK-svar]* Eventet ska innehålla de personuppgifter som hämtades för att
  visas för användaren, t.ex. inkomst och diagnos. Information som inte är personuppgifter
  ska inte loggas, med undantaget i IA-FR-04.3.
- **IA-FR-04.2** *[förslag]* Datan ligger på eventnivå och behöver ingen fastställd
  struktur i kontraktet. Ett fritt objekt (`data: {}`) räcker — eventet är tänkt för
  granskning, om det finns misstanke om att en handläggare sett något hen inte skulle se,
  och konsumenten behöver inte tolka innehållet maskinellt.
- **IA-FR-04.3** *[FK-svar]* Tills vidare får hela den hämtade payloaden ingå, även delar som
  inte är personuppgifter. Det beror på att datan ännu inte är märkt med metadata som anger
  vad som är personuppgift. Undantaget gäller bara så länge sådan metadata saknas. När den
  finns ska eventet bära enbart personuppgifterna.

### IA-FR-05 — Användare, tidpunkt och källa

- **IA-FR-05.1** *[FK-svar]* Eventet ska ange vilken användare som fick informationen visad,
  med användarens `handlaggarId`. Det finns i den token som BFF:en tar emot och
  vidarebefordrar, som `typId:varde` — samma form som `anvandare`. Det gäller dagens
  dev-token; en riktig inloggningstoken kan se annorlunda ut. Id-värdet i specens exempel är
  påhittat — se Öppna punkter.
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
  igen när specen breddades till alla användare (commit 1ab0786). FK har bekräftat att både
  processinstansen och den enskilda uppgiften behövs. Den nya handläggningsmodellen saknar
  `processinstansId`, så processinstansen anges med `handlaggningId` tillsammans med
  `uppgiftId`. Konflikten gäller därför bara GUI som laddas utan enskild uppgift.
- **IA-FR-06.2** *[1097]* Eventet ska, när det är tillämpligt, ange vilket ärende och vilken
  regel informationen visades i. Regeln identifieras med `regelId` och `regelVersion` från
  regelns `config.yaml`, inte med fritextnamn, så att loggarna går att söka entydigt.

### IA-FR-07 — Tillförlitlighet vid publicering

- **IA-FR-07.1** *[förslag]* Om ett misslyckat publiceringsförsök inte stoppar utlämnandet
  ska det framgå i den publicerande applikationens egen loggning, så att tappade händelser
  går att upptäcka i efterhand.

### IA-FR-08 — Identifierare

- **IA-FR-08.1** *[förslag]* Varje event ska ha en egen identifierare, så att konsumenten
  kan följa meddelandet genom sina egna steg. Nyttan ligger hos mottagaren; Rimfrost
  publicerar och följer inte upp. Leveransen är at-least-once, så samma event kan komma
  flera gånger, och identifieraren gör det möjligt att hantera det utan dubbla poster.

### IA-FR-09 — Topic

- **IA-FR-09.1** *[1080]* Alla event publiceras på en och samma topic. Inga subtopics, t.ex.
  per handläggningsid.
- **IA-FR-09.2** *[1080]* Topicen konfigureras statiskt, inte via dynamisk lookup.

## Icke-funktionella krav

### IA-NFR-01 — Personuppgifter

- **IA-NFR-01.1** *[FK-svar]* Eventet innehåller personuppgifter — både individernas
  identifierare och den data som visades (IA-FR-04.1) — och ska behandlas därefter i
  lagring, åtkomst och gallring.

### IA-NFR-02 — Utvecklingsbarhet

- **IA-NFR-02.1** *[förslag]* Kontraktet ska kunna utökas med mer information utan
  brytande ändringar för befintliga producenter och konsumenter.

### IA-NFR-03 — Testbarhet

- **IA-NFR-03.1** *[förslag]* Att eventet publiceras med rätt innehåll ska gå att verifiera
  i integrationstest, utan att behöva läsa producentens interna tillstånd.

## Konsekvenser för nuvarande kontrakt

Kontraktet ligger i dag på 0.2.0, där `visadInformation` (IA-FR-04) och eventets `id`
(IA-FR-08.1) har lagts till, och har inga implementationer. Ändringarna nedan följer av
kraven, men bör inte göras förrän de öppna punkterna är besvarade.

Ingen av dem är brytande. Om det visar sig att visad information måste kunna härledas till
en enskild individ i ett event som rör flera, behöver `individer` göras om till objekt som
bär datan per individ — det *är* en brytande ändring, och frågan ligger under Öppna punkter.

1. `uppgiftId` är i dag valfritt. IA-FR-06.1 gör det nödvändigt. Beslutet hänger nu på om
   portalens uppgiftslista bantas: bara om varje loggat GUI rör en enskild uppgift kan fältet
   göras obligatoriskt. FK har bekräftat att den enskilda uppgiften behövs, men inte svarat på
   vad Syfte ska innehålla när ett GUI laddas utan enskild uppgift.
2. `individer` behövs inte enligt IA-FR-03.3. Fältet kan ligga kvar som valfritt, men
   regelramverken fyller inte i det.
3. `anvandare` ska bära användarens `handlaggarId` enligt IA-FR-05.1. Värdet finns i token
   som BFF:en tar emot. Ingen ändring av kontraktet behövs.
4. Exemplet `Uppgiftslista` i `asyncapi.yaml` visar portalens uppgiftslista med
   `kalla: rimfrost-portal-bff`. Enligt IA-FR-01.1 ska den vyn loggas bara om den bär
   personuppgifter. Exemplet bör tas bort eller skrivas om när frågan om bantningen är
   avgjord.

## Öppna punkter

Kolumnen *Beslutas av* är bara ifylld där det finns belägg för det. Där den säger
"ej utrett" behöver frågan om ägarskap ställas innan frågan i sig kan drivas.

| Punkt | Berör | Beslutas av |
|---|---|---|
| Om portalens uppgiftslista ska bantas så att den inte bär personuppgifter, eller loggas som den är. `OperativUppgift` bär i dag `individer` | IA-FR-01, IA-FR-02 | Rimfrost, utifrån FK:s inriktning att hålla nere uppgiftsinformationen i OUL |
| Vad som ska stå i Syfte när ett GUI laddas utan enskild uppgift | IA-FR-06 | FK — ännu obesvarad |
| Om eventet ska gå att koppla till anropet som utlöste det, och om det redan finns ett correlation-id i anropskedjan | IA-FR-06 | Ej utrett — nyttan är främst vår egen felsökning, och vi följer inte upp efter publicering |
| Om utlämnande ska stoppas när eventet inte kan publiceras | IA-FR-07 | Ej utrett |
| Var `handlaggarId` läses: i BFF:en direkt från token, vilket kräver att FBFF-FR-03.2 skrivs om, eller i regelns backend | IA-FR-05 | Rimfrost, i FKPOC-1098 |
| Vilka `typId`-UUID:n som gäller för `handlaggarId` och personnummer | IA-FR-03 | Rimfrost |
| Om självåtkomst alls ska loggas | IA-FR-05 | Ej utrett — FK-fråga |
| Vad eventen partitioneras på | IA-FR-09 | Ej utrett |
| Övre gräns för meddelandestorlek när datan ingår i eventet | IA-FR-04 | Ej utrett |

## Besvarat av FK

Svaren nedan kommer från Försäkringskassan och ligger till
grund för kraven märkta **[FK-svar]**.

| Fråga | Svar |
|---|---|
| När utlöses loggningen? | Varje gång GUI:t laddas, när datan lämnar BFF:en |
| Ska allt som visades ligga i samma meddelande? | Ja. Datapaketet är det eller de JSON-objekt som visas i GUI:t |
| Ska OUL-listor loggas? | Inte undantagna som vy. Inriktningen är att hålla nere uppgiftsinformationen i OUL så att listan slipper bära personuppgifter |
| Vilken data ska eventet bära? | Bara personuppgifter. Hela payloaden får ingå tills data är märkt med metadata om den är personuppgift |
| Ska information som inte är personbunden loggas? | Nej, med undantaget ovan |
| Räcker det att logga vilken regel som utförs, eller behövs den enskilda uppgiftens id? | Både processinstansen och den enskilda uppgiften behövs |
| Behöver individerna listas separat? | Nej, det räcker med den data som skickas till klienten |
