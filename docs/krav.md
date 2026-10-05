# Krav — informationsaccess

Krav på det event som publiceras när information har visats för en användare.
Eventet definieras i `rimfrost-service-informationsaccess-asyncapi` och publiceras av
BFF:erna. Det kan konsumeras av t.ex. iloggning.

**Status:** utkast. Punkter märkta *öppen* är inte beslutade.

## Avgränsning

Kraven gäller eventet och dess innehåll. Utanför ligger:

- implementationen av producenten, som har egna krav i `rimfrost-framework-bff`
- den konsumerande iloggningstjänsten och hur utdrag faktiskt produceras
- behörighetskontroll, som ligger kvar hos de bakomliggande tjänsterna

## Funktionella krav

### IA-FR-01 — När eventet ska publiceras

- **IA-FR-01.1** Ett event ska publiceras varje gång information om en eller flera
  individer lämnar en BFF till klienten.
- **IA-FR-01.2** Avgörande är vad som går från BFF till klient. Information som hämtats
  internt i backend men aldrig skickats vidare ska inte loggas, och det ska inte heller
  krävas kännedom om vad klienten faktiskt renderat på skärmen.
- **IA-FR-01.3** *(öppen)* Om ett svar innehåller flera uppgifter, t.ex. en uppgiftslista,
  behöver det avgöras om det publiceras ett event per uppgift eller ett för hela svaret.
- **IA-FR-01.4** Flera BFF:er kan publicera var sitt event för det användaren upplever som
  en sammanhängande handling, t.ex. en uppgiftslista i portalen följd av att en uppgift
  öppnas i en regel. Det är avsiktligt och ska inte behandlas som dubbletter.

### IA-FR-02 — Separerbarhet per individ

- **IA-FR-02.1** Eventet ska gå att dela upp per individ, så att ett utdrag till en
  enskild person kan produceras utan att röja uppgifter om någon annan individ.
- **IA-FR-02.2** All information som rör en individ ska bäras i den individens egen
  struktur i eventet. Ingen personbunden information får ligga på eventnivå.
- **IA-FR-02.3** Information som visats och rör flera individer ska upprepas i varje
  berörd individs struktur, så att varje individs utdrag är fullständigt och fristående
  utan att behöva läsa något delat.

### IA-FR-03 — Vilka individer som ingår

- **IA-FR-03.1** Eventet ska lista de individer vars information faktiskt passerade
  BFF:en till klienten i just det svaret — inte samtliga individer som är knutna till
  ärendet.
- **IA-FR-03.2** Identifieraren för en individ finns i eventet för loggens skull. Att en
  individ förekommer i eventet ska inte tolkas som att identifieraren i sig visades för
  användaren.

### IA-FR-04 — Beskrivning av visad information

- **IA-FR-04.1** Eventet ska beskriva vilken information som visades, per individ.
- **IA-FR-04.2** *(öppen)* Granulariteten behöver beslutas: en kategori, en uppräkning av
  fältnamn, eller en referens till det som returnerades. Avvaktar struktur från Jonas.
- **IA-FR-04.3** Beskrivningen ska inte i sig innehålla de visade värdena, bara vilken
  information det rörde sig om.

### IA-FR-05 — Användare, tidpunkt och källa

- **IA-FR-05.1** Eventet ska ange vilken användare som fick informationen visad.
- **IA-FR-05.2** Eventet ska ange när informationen visades.
- **IA-FR-05.3** Eventet ska ange vilken applikation som visade informationen och
  publicerade eventet.
- **IA-FR-05.4** Användaren behöver inte vara en handläggare. Om en person tar del av sina
  egna uppgifter i en självbetjäningstjänst är användaren samma person som den individ
  informationen rör, och eventet ska publiceras på samma sätt.

### IA-FR-06 — Sammanhang

- **IA-FR-06.1** Eventet ska, när det är tillämpligt, ange vilket ärende, vilken uppgift
  och vilken regel informationen visades i. Fälten är valfria eftersom alla vägar inte har
  alla delar.
- **IA-FR-06.2** Eventet bör gå att koppla till det anrop som utlöste det, så att en
  loggpost kan härledas till en specifik begäran vid felsökning och granskning.

### IA-FR-07 — Tillförlitlighet vid publicering

- **IA-FR-07.1** *(öppen)* Det behöver beslutas vad som ska hända om eventet inte kan
  publiceras. Antingen får informationen inte lämnas ut när den inte kan loggas, eller så
  accepteras att utlämnandet sker och att loggposten kan gå förlorad. Valet påverkar både
  producentens implementation och vilka garantier loggen kan ge.
- **IA-FR-07.2** Om ett misslyckat publiceringsförsök inte stoppar utlämnandet ska det
  framgå i den publicerande applikationens egen loggning, så att tappade händelser går att
  upptäcka i efterhand.

### IA-FR-08 — Identitet och dubbletter

- **IA-FR-08.1** Varje event ska ha en egen identifierare, så att konsumenter kan skilja
  två separata visningar från samma visning levererad mer än en gång.
- **IA-FR-08.2** Konsumenter ska kunna behandla eventströmmen idempotent. Eftersom
  leveransen är at-least-once kan samma event komma flera gånger, och det får inte leda
  till dubbla poster i ett registerutdrag.

## Icke-funktionella krav

### IA-NFR-01 — Personuppgifter

- **IA-NFR-01.1** Eventet innehåller personuppgifter, bland annat individernas
  identifierare, och ska behandlas därefter i lagring, åtkomst och gallring.

### IA-NFR-02 — Utvecklingsbarhet

- **IA-NFR-02.1** Kontraktet ska kunna utökas med mer information per individ utan
  brytande ändringar för befintliga producenter och konsumenter.

### IA-NFR-03 — Testbarhet

- **IA-NFR-03.1** Att eventet publiceras med rätt innehåll ska gå att verifiera i
  integrationstest, utan att behöva läsa producentens interna tillstånd.

## Konsekvenser för nuvarande kontrakt

Kontraktet ligger i dag på 0.1.0 och har inga implementationer. Tre ändringar följer av
kraven ovan:

1. `individer` är i dag en array av `Idtyp`. För att uppfylla IA-FR-02.2 behöver posterna
   vara objekt som bär både identiteten och den individens visade information.
2. `visadInformation` är i dag reserverat på eventnivå. Enligt IA-FR-02.2 hör det i
   stället hemma per individ.
3. Eventet saknar en egen identifierare. IA-FR-08.1 kräver ett sådant fält.

## Öppna punkter

| Punkt | Krav |
|---|---|
| Ett event per uppgift eller ett per svar | IA-FR-01.3 |
| Granulariteten i beskrivningen av visad information | IA-FR-04.2 |
| Hur information som inte är personbunden ska hanteras, om sådan ska loggas alls | IA-FR-02.2 |
| Om utlämnande ska stoppas när eventet inte kan publiceras | IA-FR-07.1 |
