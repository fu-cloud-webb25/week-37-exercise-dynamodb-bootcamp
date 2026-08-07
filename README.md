# Vecka 37: DynamoDB Bootcamp – Streaming Service

I denna övning ska du bygga ett API för en mindre streamingtjänst med hjälp av **DynamoDB**, **Lambda**, **API Gateway** och **Serverless Framework**.

Du har redan arbetat med Lambda, API Gateway och Serverless Framework. Det nya i denna övning är därför **DynamoDB**.

Vårt färdiga flöde kommer se ut så här:

```text id="b90b9n"
Insomnia
   │
   ▼
API Gateway
   │
   ▼
Lambda
   │
   ▼
DynamoDB
```

Vi kommer bygga tjänsten steg för steg och testa våra endpoints i exempelvis **Insomnia**.

---

# Del 1 – Vår data

Streaming-tjänsten ska innehålla:

```text id="g6lgbp"
Series
└── Season
    └── Episode
```

Exempel:

```text id="0u1jqx"
The Last of Us
│
├── Season 1
│   ├── Episode 1
│   ├── Episode 2
│   └── Episode 3
│
└── Season 2
    ├── Episode 1
    └── Episode 2
```

I en traditionell relationsdatabas hade vi kanske skapat separata tabeller för:

```text id="2mbflz"
Series
Seasons
Episodes
```

I denna övning ska vi istället lagra all data i **en enda DynamoDB-tabell**.

Detta brukar kallas:

> **Single Table Design**

---

# Del 2 – Skapa DynamoDB-tabellen

Gå till **DynamoDB** i AWS Console och skapa en ny tabell.

Döp tabellen till:

```text id="tbfmwh"
streaming-db
```

Tabellen ska ha två nycklar:

```text id="0ovhga"
Partition key: PK
Sort key:      SK
```

Båda ska vara av typen:

```text id="v4p87j"
String
```

I den här första övningen skapar vi tabellen manuellt för att tydligt kunna se hur DynamoDB fungerar.

Senare kommer vi skapa våra DynamoDB-tabeller direkt genom `serverless.yml`.

---

# Del 3 – Hur ska våra nycklar se ut?

Vi ska använda läsbara nycklar där det tydligt framgår vilken typ av data ett ID representerar.

Grundprincipen är:

```text id="yav5f6"
TYPE:id
```

Exempel:

```text id="q08tr8"
SERIES:tlou
```

När vår data innehåller flera nivåer separerar vi dem med `#`.

Exempel:

```text id="clyif9"
SEASON:1#EPISODE:3
```

Vi använder alltså:

```text id="taz4na"
:
```

mellan **typ och ID**, och:

```text id="nt84hd"
#
```

mellan olika **nivåer**.

Det gör våra nycklar relativt enkla att läsa.

---

# Del 4 – Lägg in vår första serie

Innan vi börjar programmera behöver vi lite testdata.

Skapa följande item manuellt i DynamoDB:

```json id="40uj97"
{
  "PK": "SERIES:tlou",
  "SK": "SERIES:tlou",
  "type": "series",
  "title": "The Last of Us",
  "genre": "Drama",
  "releaseYear": 2023
}
```

Detta item representerar själva serien.

Nycklarna är:

```text id="bl3dbf"
PK = SERIES:tlou
SK = SERIES:tlou
```

---

# Del 5 – Bygg vår första endpoint

Nu ska vi direkt koppla DynamoDB till vårt API.

Vi vill kunna göra:

```http id="nyxx2z"
GET /series/tlou
```

och få tillbaka informationen om *The Last of Us* från DynamoDB.

Skapa ett Serverless Framework-projekt och installera DynamoDB-klienterna:

```bash id="x0hlzj"
npm install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
```

Skapa sedan en DynamoDB Document Client som dina Lambda-funktioner kan använda:

```js id="i7g0cb"
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient } from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});

export const db = DynamoDBDocumentClient.from(client);
```

---

# Del 6 – GetCommand

Skapa en Lambda-funktion:

```text id="vkg26e"
getSeries
```

Funktionen ska läsa `seriesId` från API Gateways path parameter.

När vi anropar:

```http id="c3uwlb"
GET /series/tlou
```

får vi:

```text id="1mr4v7"
seriesId = tlou
```

Funktionen kan då skapa nycklarna:

```text id="3v0p2x"
PK = SERIES:tlou
SK = SERIES:tlou
```

Använd `GetCommand` från:

```js id="hnwwk9"
@aws-sdk/lib-dynamodb
```

för att hämta objektet från DynamoDB.

Din funktion behöver alltså göra ungefär:

```text id="f4t5hp"
seriesId
   ↓
SERIES:tlou
   ↓
GetCommand
   ↓
DynamoDB
   ↓
Series
```

---

# Del 7 – Koppla API Gateway

Konfigurera funktionen i `serverless.yml` så att den triggas av:

```http id="ux2eyy"
GET /series/{seriesId}
```

Glöm inte att din Lambda-funktion behöver rättigheter att läsa från DynamoDB.

Deploya projektet:

```bash id="m6pvw4"
serverless deploy
```

Öppna sedan Insomnia och testa:

```http id="e2is7q"
GET <din-api-url>/series/tlou
```

Om allt fungerar ska du få tillbaka:

```json id="4q3d43"
{
  "PK": "SERIES:tlou",
  "SK": "SERIES:tlou",
  "type": "series",
  "title": "The Last of Us",
  "genre": "Drama",
  "releaseYear": 2023
}
```

Du har nu byggt hela kedjan:

```text id="vs1l6j"
Insomnia
   ↓
API Gateway
   ↓
getSeries
   ↓
GetCommand
   ↓
DynamoDB
```

---

# Del 8 – Lägg till en säsong

Nu ska vi börja bygga vår hierarki.

Lägg manuellt till följande item i DynamoDB:

```json id="h6g6vh"
{
  "PK": "SERIES:tlou",
  "SK": "SEASON:1",
  "type": "season",
  "seasonNumber": 1,
  "title": "Season 1"
}
```

Notera att serien och säsongen har samma:

```text id="qptm3r"
PK = SERIES:tlou
```

men olika Sort Keys:

```text id="hkmk15"
SERIES:tlou
SEASON:1
```

Det innebär att båda objekten tillhör samma **partition**.

---

# Del 9 – Lägg till avsnitt

Lägg nu manuellt till några avsnitt.

Ett avsnitt använder följande struktur:

```text id="nsp9bp"
PK = SERIES:<seriesId>
SK = SEASON:<season>#EPISODE:<episode>
```

Exempel:

```json id="5ce7wi"
{
  "PK": "SERIES:tlou",
  "SK": "SEASON:1#EPISODE:1",
  "type": "episode",
  "seasonNumber": 1,
  "episodeNumber": 1,
  "title": "When You're Lost in the Darkness",
  "duration": 81
}
```

Lägg även till:

```json id="qmxhzg"
{
  "PK": "SERIES:tlou",
  "SK": "SEASON:1#EPISODE:2",
  "type": "episode",
  "seasonNumber": 1,
  "episodeNumber": 2,
  "title": "Infected",
  "duration": 53
}
```

och minst ett ytterligare avsnitt.

Din partition innehåller nu ungefär:

```text id="8f70cz"
SERIES:tlou
│
├── SERIES:tlou
├── SEASON:1
├── SEASON:1#EPISODE:1
├── SEASON:1#EPISODE:2
└── SEASON:1#EPISODE:3
```

---

# Del 10 – QueryCommand

Nu vill vi skapa:

```http id="6m6ff4"
GET /series/tlou/seasons/1
```

Endpointen ska returnera alla avsnitt från säsong 1.

Här räcker inte längre `GetCommand`.

`GetCommand` används när vi känner till hela nyckeln till ett specifikt item.

Nu vill vi istället fråga efter **flera items som hör ihop**.

Därför använder vi:

```js id="96sf19"
QueryCommand
```

Vi vill göra följande query:

```text id="9crfy1"
PK = SERIES:tlou

AND

SK begins_with "SEASON:1#EPISODE:"
```

Det gör att DynamoDB endast returnerar avsnitten från säsong 1.

---

# Del 11 – Skapa getEpisodes

Skapa Lambda-funktionen:

```text id="ux1i69"
getEpisodes
```

Den ska läsa:

```text id="lv8tzj"
seriesId
season
```

från path parameters.

Konfigurera sedan endpointen:

```http id="bd9o0r"
GET /series/{seriesId}/seasons/{season}
```

Använd `QueryCommand` och `begins_with` för att hämta rätt data.

Deploya och testa:

```http id="d83zd1"
GET <din-api-url>/series/tlou/seasons/1
```

Du ska få tillbaka avsnitten från säsong 1.

---

# Del 12 – Lägg till säsong 2

Lägg manuellt till:

```text id="llrvby"
PK = SERIES:tlou
SK = SEASON:2
```

och minst tre avsnitt:

```text id="p0z60i"
SEASON:2#EPISODE:1
SEASON:2#EPISODE:2
SEASON:2#EPISODE:3
```

Testa sedan:

```http id="nvbfbl"
GET /series/tlou/seasons/1
```

och:

```http id="eqr0jc"
GET /series/tlou/seasons/2
```

Kontrollera att respektive request endast returnerar avsnitten från rätt säsong.

---

# Del 13 – Hämta hela serien

Nu vill vi kunna göra:

```http id="dfxys6"
GET /series/tlou/content
```

och få tillbaka **allt som tillhör serien**.

Eftersom alla våra items delar:

```text id="3z5c95"
PK = SERIES:tlou
```

kan vi använda `QueryCommand` för att hämta hela partitionen.

Skapa funktionen:

```text id="p2ht03"
getSeriesContent
```

och endpointen:

```http id="9l1r0h"
GET /series/{seriesId}/content
```

Resultatet ska innehålla:

```text id="nhv96l"
Series
Season 1
Episode 1
Episode 2
Episode 3
Season 2
Episode 1
Episode 2
Episode 3
```

Här ser du en av anledningarna till att våra objekt delar samma Partition Key.

---

# Del 14 – Våra första Access Patterns

Stanna upp och titta på vad vi faktiskt har gjort.

Vår databas är designad för att effektivt kunna svara på vissa frågor.

Dessa brukar kallas:

> **Access Patterns**

Vi kan nu:

### Hämta en specifik serie

```text id="0i79gu"
PK = SERIES:tlou
SK = SERIES:tlou
```

### Hämta hela serien

```text id="f44j6u"
PK = SERIES:tlou
```

### Hämta alla avsnitt från säsong 1

```text id="kx9wqe"
PK = SERIES:tlou
SK begins_with "SEASON:1#EPISODE:"
```

Det är därför våra nycklar ser ut som de gör.

I DynamoDB funderar vi på **hur applikationen behöver läsa datan** och designar våra nycklar därefter.

---

# Del 15 – PutCommand

Hittills har vi lagt till vår data manuellt.

Nu ska API:t kunna göra det.

Skapa:

```http id="vd6fzb"
POST /series/{seriesId}/seasons/{season}/episodes
```

Request body:

```json id="d59h6d"
{
  "episodeNumber": 4,
  "title": "My New Episode",
  "duration": 58
}
```

Din Lambda-funktion ska använda:

```js id="ztc19k"
PutCommand
```

för att skapa ett nytt item.

Med:

```text id="ok3ejv"
seriesId = tlou
season = 1
episodeNumber = 4
```

ska nycklarna bli:

```text id="t89id8"
PK = SERIES:tlou
SK = SEASON:1#EPISODE:4
```

Testa POST-anropet i Insomnia.

Kör sedan:

```http id="09y2zi"
GET /series/tlou/seasons/1
```

och kontrollera att det nya avsnittet finns med.

Kontrollera även DynamoDB-tabellen i AWS Console.

---

# Del 16 – UpdateCommand

Nu ska ett befintligt avsnitt kunna uppdateras.

Skapa:

```http id="hdj0x7"
PUT /series/{seriesId}/seasons/{season}/episodes/{episode}
```

Exempel:

```http id="q6nifn"
PUT /series/tlou/seasons/1/episodes/4
```

Request body:

```json id="p4jcr5"
{
  "title": "A Much Better Episode",
  "duration": 62
}
```

Använd:

```js id="f5gz97"
UpdateCommand
```

för att uppdatera objektet.

För att identifiera rätt item behöver du:

```text id="a7u5l1"
PK = SERIES:tlou
SK = SEASON:1#EPISODE:4
```

Testa sedan GET-endpointen igen och kontrollera att informationen har ändrats.

---

# Del 17 – DeleteCommand

Sista steget är att kunna ta bort ett avsnitt.

Skapa:

```http id="qkpfo9"
DELETE /series/{seriesId}/seasons/{season}/episodes/{episode}
```

Exempel:

```http id="9c3gfn"
DELETE /series/tlou/seasons/1/episodes/4
```

Använd:

```js id="np94yg"
DeleteCommand
```

med:

```text id="j5x11d"
PK = SERIES:tlou
SK = SEASON:1#EPISODE:4
```

Kontrollera sedan med GET-endpointen att avsnittet inte längre finns.

---

# Del 18 – CRUD och DynamoDB

Du har nu arbetat med fyra centrala DynamoDB-operationer:

| Command         | Användning                   |
| --------------- | ---------------------------- |
| `GetCommand`    | Hämta ett specifikt item     |
| `QueryCommand`  | Hämta flera relaterade items |
| `PutCommand`    | Skapa/ersätt ett item        |
| `UpdateCommand` | Uppdatera ett item           |
| `DeleteCommand` | Ta bort ett item             |

Du har dessutom byggt följande endpoints:

| Method   | Endpoint                                                 | DynamoDB        |
| -------- | -------------------------------------------------------- | --------------- |
| `GET`    | `/series/{seriesId}`                                     | `GetCommand`    |
| `GET`    | `/series/{seriesId}/content`                             | `QueryCommand`  |
| `GET`    | `/series/{seriesId}/seasons/{season}`                    | `QueryCommand`  |
| `POST`   | `/series/{seriesId}/seasons/{season}/episodes`           | `PutCommand`    |
| `PUT`    | `/series/{seriesId}/seasons/{season}/episodes/{episode}` | `UpdateCommand` |
| `DELETE` | `/series/{seriesId}/seasons/{season}/episodes/{episode}` | `DeleteCommand` |

---

# Del 19 – Skapa en egen serie

Nu ska du lägga till en helt egen serie.

Den kan vara verklig eller påhittad.

Men denna gång får du **inte lägga in datan manuellt i DynamoDB**.

Använd ditt API.

Din serie ska innehålla:

* information om serien
* minst två säsonger
* minst tre avsnitt per säsong

Du kommer därför behöva fundera på vilken funktionalitet ditt API fortfarande saknar.

Exempelvis:

```http id="rpkvny"
POST /series
```

och:

```http id="2fzf59"
POST /series/{seriesId}/seasons
```

Designa och implementera dessa endpoints själv.

När du är klar ska hela serien ha skapats genom ditt API.

---

# Del 20 – Testa din streamingtjänst

Kontrollera att du kan:

1. Skapa en serie.
2. Skapa säsonger.
3. Skapa avsnitt.
4. Hämta information om serien.
5. Hämta hela seriens innehåll.
6. Hämta avsnitten från en specifik säsong.
7. Uppdatera ett avsnitt.
8. Ta bort ett avsnitt.

Kontrollera även din data i DynamoDB Console.

Du har nu byggt:

```text id="m4pk5g"
             Insomnia
                 │
                 ▼
            API Gateway
                 │
                 ▼
              Lambda
                 │
                 ▼
        ┌─────────────────┐
        │    DynamoDB     │
        │                 │
        │ SERIES          │
        │   └─ SEASON     │
        │       └─ EPISODE│
        └─────────────────┘
```

---

# Level Up – Registrering och inloggning

Vill du förbereda dig lite extra inför kommande examination kan du bygga ut streamingtjänsten med **användare, registrering och inloggning**.

Målet är att en användare ska kunna:

```text
Registrera sig
      ↓
Logga in
      ↓
Få en JWT
      ↓
Anropa en skyddad endpoint
```

---

## Steg 1 – Lagra användare

Vår DynamoDB-tabell innehåller redan serier, säsonger och avsnitt.

Nu ska den även kunna innehålla:

```text
USER
```

Fundera först på hur en användare skulle kunna representeras med vår nuvarande nyckelstruktur.

Ett exempel skulle kunna vara:

```text
PK = USER:anna@example.com
SK = USER:anna@example.com
```

Ett user-item skulle exempelvis kunna se ut så här:

```json
{
  "PK": "USER:anna@example.com",
  "SK": "USER:anna@example.com",
  "type": "user",
  "username": "anna",
  "email": "anna@example.com",
  "password": "<hashed password>",
  "createdAt": "2026-08-07T10:00:00.000Z"
}
```

> **Viktigt:** Ett lösenord ska aldrig sparas som vanlig text i databasen.

---

## Steg 2 – Registrering

Skapa endpointen:

```http
POST /auth/register
```

Request body:

```json
{
  "username": "anna",
  "email": "anna@example.com",
  "password": "mySecretPassword"
}
```

Din Lambda-funktion ska:

1. Kontrollera att nödvändig data finns.
2. Kontrollera om användaren redan existerar.
3. Hasha lösenordet med `bcrypt`.
4. Skapa användaren i DynamoDB.
5. Returnera ett lämpligt svar.

Installera exempelvis:

```bash
npm install bcryptjs
```

> `bcryptjs` kan vara enklare att använda i Lambda än paket som innehåller plattformsspecifika binärer.

Lösenordet som sparas i DynamoDB ska alltså **inte** vara:

```text
mySecretPassword
```

utan ett hashat värde.

---

## Steg 3 – Inloggning

Skapa endpointen:

```http
POST /auth/login
```

Request body:

```json
{
  "email": "anna@example.com",
  "password": "mySecretPassword"
}
```

Din funktion ska:

1. Hämta användaren från DynamoDB.
2. Kontrollera lösenordet mot det hashade lösenordet.
3. Returnera fel om användaren eller lösenordet är felaktigt.
4. Skapa en JWT om inloggningen lyckas.

Installera:

```bash
npm install jsonwebtoken
```

Ett lyckat svar kan exempelvis innehålla:

```json
{
  "token": "eyJhbGciOiJIUzI1NiIs..."
}
```

Token ska innehålla information som gör att användaren kan identifieras, exempelvis:

```json
{
  "userId": "anna@example.com",
  "username": "anna"
}
```

---

## Steg 4 – JWT Secret

Hemligheten som används för att signera JWT-token ska **inte** skrivas direkt i JavaScript-koden.

Använd istället en environment variable.

Exempel i `serverless.yml`:

```yaml
provider:
  environment:
    JWT_SECRET: ${env:JWT_SECRET}
```

I din Lambda kan den sedan hämtas med:

```js
process.env.JWT_SECRET
```

Lägg aldrig JWT-hemligheten i GitHub.

---

## Steg 5 – Skapa en skyddad endpoint

Skapa nu en endpoint som endast ska gå att använda av en inloggad användare.

Exempel:

```http
GET /users/me
```

Requesten ska innehålla:

```http
Authorization: Bearer <token>
```

Din Lambda ska:

1. Hämta token från `Authorization`-headern.
2. Verifiera token.
3. Hämta informationen om användaren.
4. Returnera användaren.

Om token saknas eller är ogiltig ska API:t returnera:

```text
401 Unauthorized
```

---

## Extra Level Up – My List

Vill du bygga vidare ytterligare kan varje användare få en egen lista med serier som hen vill titta på.

Exempel:

```http
POST /users/me/watchlist
```

Request body:

```json
{
  "seriesId": "tlou"
}
```

Fundera på hur detta skulle kunna representeras i vår single-table design.

Ett alternativ skulle exempelvis kunna börja med:

```text
PK = USER:anna@example.com
SK = WATCHLIST:SERIES:tlou
```

Då skulle alla objekt som tillhör användaren kunna ligga tillsammans:

```text
USER:anna@example.com
│
├── USER:anna@example.com
├── WATCHLIST:SERIES:tlou
├── WATCHLIST:SERIES:dark
└── WATCHLIST:SERIES:fallout
```

Fundera på vilken Query som skulle behövas för att hämta hela användarens watchlist.
