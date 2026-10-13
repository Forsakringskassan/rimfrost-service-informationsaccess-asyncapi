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
| **[1097]** | Redan beslutat i PR #1 och infört i `asyncapi.yaml`. |
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
- **IA-FR-01.3** *[förslag]* Det användaren upplever som en sammanhängande handling kan ge
  flera event, publicerade från olika steg och av olika anledningar. Det är avsiktligt och
  ska inte behandlas som dubbletter.

> **Antagande, obekräftat:** loggningen utlöses när en handläggare öppnar en uppgift, inte
> när en uppgiftslista visas. Det stämmer med att FKPOC-1080 bara beskriver "en manuell
> uppgifts GUI", men behöver bekräftas — se Öppna punkter.

### IA-FR-02 — Ett event per visning

Enligt FKPOC-1080 sker uppdelningen per individ hos Mimer, genom kopplingen uppgift →
process → ärendedata. Det är alltså inget Rimfrost behöver lösa i kontraktet, och ett
registerutdrag per individ kan produceras utan att eventet delas upp.

- **IA-FR-02.1** *[1080]* Allt som visades vid ett tillfälle ska ligga i samma meddelande.
  Eventet ska inte delas upp i flera, varken per individ eller per uppgift. FKPOC-1080:
  *"ett datapaket på en Kafkatopic som Mimer konsumerar."*

> **Rationale:** ett tidigare utkast krävde ett event per individ. Det är struket. En
> uppdelning gör den totala bilden av vad som visades vid en viss tidpunkt svårare att
> sammanställa, en konsument kan inte veta att den fått alla delar utan att eventet bär ett
> förväntat antal, och det finns ingen väg att be om meddelande 5 av 7 igen.

> **Kontrollerat:** ett regelsvar rör en individ i taget.
> `rimfrost-regel-rtf-manuell-bff` och `rimfrost-regel-bekraftabeslut-bff` returnerar båda
> `kund` i singular, utan lista. Det gäller **inte** portalens uppgiftslista —
> `OperativUppgift` i `rimfrost-portal-bff` bär `individer` som lista. Den vyn är parkerad,
> se Öppna punkter.

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
- **IA-FR-04.2** *[förslag]* Datan ligger på eventnivå och behöver ingen fastställd
  struktur i kontraktet. Ett fritt objekt (`data: {}`) räcker — eventet är tänkt för
  granskning, om det finns misstanke om att en handläggare sett något hen inte skulle se,
  och konsumenten behöver inte tolka innehållet maskinellt.

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

- **IA-NFR-01.1** *[1080]* Eventet innehåller personuppgifter — både individernas
  identifierare och den data som visades (IA-FR-04.1) — och ska behandlas därefter i
  lagring, åtkomst och gallring.

### IA-NFR-02 — Utvecklingsbarhet

- **IA-NFR-02.1** *[förslag]* Kontraktet ska kunna utökas med mer information utan
  brytande ändringar för befintliga producenter och konsumenter.

### IA-NFR-03 — Testbarhet

- **IA-NFR-03.1** *[förslag]* Att eventet publiceras med rätt innehåll ska gå att verifiera
  i integrationstest, utan att behöva läsa producentens interna tillstånd.

## Konsekvenser för nuvarande kontrakt

Kontraktet ligger i dag på 0.1.0 och har inga implementationer. Ändringarna nedan följer av
kraven, men bör inte göras förrän de öppna punkterna är besvarade.

Ingen av dem är brytande. Om det visar sig att visad information måste kunna härledas till
en enskild individ i ett event som rör flera, behöver `individer` göras om till objekt som
bär datan per individ — det *är* en brytande ändring, och frågan ligger under Öppna punkter.

1. `visadInformation` är uppskjutet till FKPOC-1103 enligt beslutet i PR #1. Enligt
   IA-FR-04.1 ska det bära den faktiska datan, och enligt IA-FR-04.2 ligger det på eventnivå
   som ett fritt objekt. Att lägga till fältet som valfritt bryter inte befintliga
   konsumenter.
2. Eventet saknar en egen identifierare. IA-FR-08.1 kräver ett sådant fält, så att
   konsumenten kan följa meddelandet genom sina egna steg. Ett nytt valfritt fält bryter
   inte befintliga konsumenter.
3. `uppgiftId` är i dag valfritt. IA-FR-06.1 gör det nödvändigt. Beslutet hänger ihop med
   frågan om vyer utan uppgift ska ingå — i så fall behöver Mimer något annat att koppla på.
4. `individer` har i dag `minItems: 1` utan övre gräns, och det ska den fortsätta ha.
   Eftersom ett event bär allt som visades (IA-FR-02.1) kan ett svar som rör flera individer
   listas i samma event. Ingen ändring behövs.
5. `anvandare` behöver bekräftas bära kortnummer enligt IA-FR-05.1. Verifieras mot
   `anropare` i FKPOC-1098 och FKPOC-1103.

## Öppna punkter

Kolumnen *Beslutas av* är bara ifylld där det finns belägg för det. Där den säger
"ej utrett" behöver frågan om ägarskap ställas innan frågan i sig kan drivas.

| Punkt | Berör | Beslutas av |
|---|---|---|
| Vyer utan uppgift, t.ex. OUL-listor. Loggas eventuellt först när en uppgift öppnas, se antagandet under IA-FR-01 | IA-FR-01, IA-FR-06 | FK |
| Om FK vill ha allt i samma meddelande. Rimfrost läser FKPOC-1080 så, men formuleringen kan tolkas olika | IA-FR-02 | FK |
| Om en BFF-respons kan visa uppgifter om flera individer | IA-FR-02 | Besvarat för regler: nej. Öppet för uppgiftslistan, som är parkerad |
| Om eventet ska gå att koppla till anropet som utlöste det, och om det redan finns ett correlation-id i anropskedjan | IA-FR-06 | Ej utrett — nyttan är främst vår egen felsökning, och vi följer inte upp efter publicering |
| Om information som inte är personbunden ska loggas | IA-FR-02 | Ej utrett |
| Hur mycket av den visade datan eventet får bära — hela paketet eller något reducerat, och om personuppgifter får ingå rakt av | IA-FR-04, IA-NFR-01 | FK |
| Om utlämnande ska stoppas när eventet inte kan publiceras | IA-FR-07 | Ej utrett |
| Om kortnummer finns i Rimfrosts anroparkontext | IA-FR-05 | Rimfrost, i FKPOC-1098/1103 |
| Vilka `typId`-UUID:n som gäller för kortnummer och personnummer | IA-FR-03 | Rimfrost |
| Om självåtkomst alls ska loggas | IA-FR-05 | Ej utrett — FK-fråga |
| Vad eventen partitioneras på | IA-FR-09 | Ej utrett |
| Övre gräns för meddelandestorlek när datan ingår i eventet | IA-FR-04 | Ej utrett |
