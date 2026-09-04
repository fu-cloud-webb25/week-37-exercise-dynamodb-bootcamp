# Vecka 37: DynamoDB Bootcamp – Streaming Service

# Övning 1 – Koppla DynamoDB till ett befintligt API

Under förra veckan har du byggt API:er med:

```text
Lambda
API Gateway
Serverless Framework
```

Du har bland annat arbetat med **Shakespearean Insults API**, och även fått möjlighet att skapa ett eget API.

Hittills har datan i våra API:er antingen varit hårdkodad eller lagrats direkt i våra Lambda-funktioner.

Problemet med detta är att datan **inte är persistent**.

När Lambda-funktionen startas om kan datan försvinna.

Nu ska vi lösa det genom att koppla vårt API till:

```text
DynamoDB
```

Vårt flöde kommer nu se ut så här:

```text
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

---

# Del 1 – Välj API

Utgå från ett API som du byggde under förra veckan.

Du kan exempelvis använda:

```text
Shakespearean Insults API
```

eller något av de egna API:er du byggt.

Målet är att **behålla ditt befintliga API**, men ersätta den tidigare datalagringen med DynamoDB.

I exemplen nedan kommer vi utgå från Shakespearean Insults API.

---

# Del 2 – Skapa en DynamoDB-tabell

Gå till **DynamoDB** i AWS Console.

Skapa en ny tabell.

Exempel:

```text
Table name:
insults-db
```

Skapa följande Partition Key:

```text
id
```

Datatyp:

```text
String
```

Vi behöver ingen Sort Key i den här övningen.

Ett item i vår tabell skulle exempelvis kunna se ut så här:

```json
{
  "id": "a8f3d2",
  "insult": "Thou art a boil, a plague-sore!"
}
```

Om du arbetar med ditt eget API behöver du själv fundera på:

* Vad ska tabellen heta?
* Vad ska vara Partition Key?
* Vilka egenskaper behöver varje item innehålla?

---

# Del 3 – Installera DynamoDB-klienterna

Öppna ditt befintliga Serverless Framework-projekt.

Installera:

```bash
npm install @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
```

Skapa sedan en DynamoDB Document Client som dina Lambda-funktioner kan använda.

Exempel:

```js
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient } from "@aws-sdk/lib-dynamodb";

const client = new DynamoDBClient({});

export const db = DynamoDBDocumentClient.from(client);
```

Nu kan våra Lambda-funktioner kommunicera med DynamoDB.

---

# Del 4 – PutCommand

Vi börjar med att skapa data.

Vi vill exempelvis kunna göra:

```http
POST /insults
```

Request body:

```json
{
  "insult": "Thou art a boil, a plague-sore!"
}
```

Lambda-funktionen ska skapa ett unikt ID och spara objektet i DynamoDB.

Resultatet skulle exempelvis kunna bli:

```json
{
  "id": "a8f3d2",
  "insult": "Thou art a boil, a plague-sore!"
}
```

För att skapa ett item använder vi:

```js
PutCommand
```

från:

```js
@aws-sdk/lib-dynamodb
```

Flödet blir:

```text
POST /insults
      │
      ▼
Lambda
      │
      ▼
skapa ID
      │
      ▼
PutCommand
      │
      ▼
DynamoDB
```

Implementera funktionen och testa den med Insomnia.

Kontrollera även i DynamoDB Console att ditt item har skapats.

---

# Del 5 – GetCommand

Nu ska vi kunna hämta ett specifikt item.

Vi vill kunna göra:

```http
GET /insults/{id}
```

Exempel:

```http
GET /insults/a8f3d2
```

Eftersom vi känner till objektets Partition Key kan vi använda:

```js
GetCommand
```

Funktionen behöver alltså göra ungefär:

```text
id
 ↓
GetCommand
 ↓
DynamoDB
 ↓
Insult
```

Testa endpointen med ett ID som finns i databasen.

Fundera även på:

> Vad bör API:t returnera om ID:t inte finns?

---

# Del 6 – Hämta alla insults

Nu vill vi kunna göra:

```http
GET /insults
```

Här uppstår ett nytt problem.

Med `GetCommand` behöver vi känna till:

```text
id
```

Men i det här fallet vill vi hämta **alla items i tabellen**.

Vi känner alltså inte till deras ID:n.

För detta kan vi använda:

```js
ScanCommand
```

Ett Scan läser igenom tabellen och returnerar de items som finns där.

Implementera:

```http
GET /insults
```

med hjälp av:

```js
ScanCommand
```

Testa sedan endpointen i Insomnia.

Du bör nu få tillbaka alla insults som finns i DynamoDB.

> **Fundera:** Vad tror du händer om tabellen innehåller 10 items? 1 000 items? 1 000 000 items?

Vi kommer återkomma till detta senare.

---

# Del 7 – UpdateCommand

Nu ska ett befintligt item kunna uppdateras.

Exempel:

```http
PUT /insults/{id}
```

Request body:

```json
{
  "insult": "Thou art an updated insult!"
}
```

För att uppdatera ett item kan vi använda:

```js
UpdateCommand
```

Lambda-funktionen behöver veta vilket item som ska uppdateras genom dess:

```text
id
```

Implementera endpointen.

Testa sedan:

```http
GET /insults/{id}
```

och kontrollera att informationen har ändrats.

Kontrollera även resultatet i DynamoDB Console.

---

# Del 8 – DeleteCommand

Sista CRUD-operationen är att ta bort ett item.

Skapa:

```http
DELETE /insults/{id}
```

För att ta bort objektet använder vi:

```js
DeleteCommand
```

Testa först att objektet finns:

```http
GET /insults/{id}
```

Ta sedan bort det:

```http
DELETE /insults/{id}
```

och försök därefter hämta det igen.

Vad bör API:t returnera?

---

# Del 9 – CRUD med DynamoDB

Du har nu arbetat med fem DynamoDB-operationer:

| Command         | Användning                    |
| --------------- | ----------------------------- |
| `GetCommand`    | Hämta ett specifikt item      |
| `ScanCommand`   | Läsa items från hela tabellen |
| `PutCommand`    | Skapa eller ersätta ett item  |
| `UpdateCommand` | Uppdatera ett item            |
| `DeleteCommand` | Ta bort ett item              |

Ditt API använder nu:

```text
              Insomnia
                  │
                  ▼
             API Gateway
                  │
                  ▼
                Lambda
                  │
                  ▼
             ┌──────────┐
             │ DynamoDB │
             └──────────┘
```

Datan ligger alltså inte längre hårdkodad i Lambda-funktionen.

---

# Del 10 – Gör ditt API persistent

Om du hittills följt exemplen med Shakespearean Insults API är det nu dags att kontrollera att hela ditt API använder DynamoDB.

Du ska kunna:

1. Skapa ett nytt item.
2. Hämta ett specifikt item.
3. Hämta alla items.
4. Uppdatera ett item.
5. Ta bort ett item.

Ingen data som används av dessa endpoints ska längre vara hårdkodad i Lambda-funktionerna.

Om du istället arbetar med ditt eget API gäller samma princip.

Anpassa tabell, data och endpoints efter ditt eget projekt.

---

# Level Up – Flytta tabellen till serverless.yml

Hittills har vi skapat DynamoDB-tabellen manuellt i AWS Console.

Men vi använder redan Serverless Framework för att beskriva resten av vår infrastruktur.

Flytta därför skapandet av DynamoDB-tabellen till:

```text
serverless.yml
```

Målet är att:

```bash
serverless deploy
```

ska kunna skapa både:

```text
Lambda-funktioner
API Gateway
DynamoDB-tabell
```

Fundera även på hur Lambda-funktionerna kan få tabellens namn genom en:

```text
environment variable
```

istället för att skriva tabellnamnet direkt i JavaScript-koden.

---

# Level Up – Validering och felhantering

Bygg vidare på det du redan lärt dig om Middy och validering.

API:t ska exempelvis kunna hantera:

```text
400 Bad Request
404 Not Found
500 Internal Server Error
```

Fundera på situationer som:

* Request body saknas.
* Ett obligatoriskt fält saknas.
* Ett ID inte existerar.
* DynamoDB-anropet misslyckas.

Försök hålla Lambda-funktionerna små genom att flytta gemensam funktionalitet till middleware där det passar.

---

# Inför nästa steg

Vi kan nu lagra data i DynamoDB.

Men en av våra endpoints krävde:

```js
ScanCommand
```

för att hitta datan.

Det fungerar för vårt lilla API.

Men tänk om vår databas innehöll:

```text
1 000 000 items
```

och vi bara ville hämta en liten del av dem?

Finns det något bättre sätt att strukturera vår data?

Det är precis det vi ska undersöka i nästa steg.

---

# Övning 2 – Streaming Service och Single Table Design

Nu ska vi bygga en större tjänst där vi behöver fundera mer på **hur vår data ska läsas** innan vi bestämmer hur den ska lagras.

Vi ska bygga en streamingtjänst med:

```text
Series
└── Season
    └── Episode
```

Den här gången ska vi inte börja med tabellen.

Vi börjar istället med en fråga:

> **Vilka frågor behöver vår applikation kunna svara på?**

Detta leder oss vidare till:

```text
Access Patterns
       ↓
Partition Keys
       ↓
Sort Keys
       ↓
Single Table Design
```

Nu börjar nästa del av övningen.

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
