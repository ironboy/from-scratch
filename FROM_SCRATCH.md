# Monorepo från grunden: .NET Minimal API + Vite React TypeScript + Tester

## Förutsättningar

- **Node.js 24 LTS** (24.14.0 eller senare)
- **npm 11**
- **.NET 10 SDK** (10.0.x)
- En terminal (bash, zsh, PowerShell — spelar ingen roll)
- En kodeditor (VS Code rekommenderas)

## Vad vi ska bygga — och varför

Innan vi skriver ett enda kommando: låt oss prata om *varför* vi sätter upp det på just det här sättet.

### En enda `package.json` i projektroten

Vi vill att alla i teamet ska kunna köra:

```bash
npm install
npm run dev
```

…och att allt bara startar. Ingen jakt på vilken undermapp man ska stå i, inga `cd backend && dotnet run` i ett separat terminalfönster. **En enda plats, ett enda kommando.**

### `concurrently` — starta allt på en gång

Vår app består av två processer under utveckling:

1. **Vite dev server** — serverar React-frontenden med hot reload
2. **.NET Minimal API** — vår C#-backend

Med npm-paketet `concurrently` kan vi starta båda parallellt via ett enda npm-script. Båda processernas output syns i samma terminal, färgkodade så man ser vad som kommer varifrån.

### Vite proxy — slipp CORS helt

Under utveckling kör Vite på t.ex. `http://localhost:5173` och vår .NET-backend på `http://localhost:5001`. Det är två olika *origins* — och webbläsaren blockerar anrop mellan dem (CORS).

Man *kan* konfigurera CORS-headers i backenden. Men det är:

- Extra kod som bara behövs under utveckling
- Lätt att glömma ta bort/justera inför produktion
- En felkälla ("varför fungerar det lokalt men inte i produktion?")

**Bättre lösning:** Vi konfigurerar Vite att *proxy:a* alla anrop till `/api` vidare till vår .NET-backend. Frontenden tror att den pratar med sig själv. Ingen CORS behövs.

Och det fina: i produktion lägger man typiskt frontend och backend bakom samma domän (via en reverse proxy som nginx eller en molntjänst) — och då fungerar samma `/api`-sökvägar utan ändring. **Koden är portabel.**

### Alla tester via npm-scripts

Vi kommer att ha flera typer av tester:

| Typ | Verktyg | Testar |
|-----|---------|--------|
| Enhetstester (backend) | xUnit | C#-logik |
| API-tester | post-they + Newman | REST-endpoints |
| E2E-tester | Playwright + BDD | Hela flöden i webbläsaren |
| Komponenttester (frontend) | ViTest | React-komponenter |

Alla ska kunna köras via npm-scripts från projektroten:

```bash
npm test              # kör allt
npm run test:xunit    # bara xUnit
npm run test:api      # bara API-tester
npm run test:e2e      # bara Playwright/BDD
npm run test:vitest   # bara ViTest
```

### Minimal setup för nya teammedlemmar

Målet är att en ny utvecklare i teamet bara behöver:

```bash
git clone <repo-url>
cd projektnamn
npm install
dotnet restore backend/MyApp.slnx
npm start
```

Fem kommandon. Klart. Inga manuella steg, ingen dokumentation att missa.

---

## Steg 1: Scaffolda frontend med Vite

Skapa en ny tom mapp för projektet och öppna den i VS Code. Öppna sedan terminalen i VS Code (`Ctrl+ö` / `` Ctrl+` ``) och kör:

```bash
npm create vite@latest . -- --template react-ts
```

Vi använder `.` (punkt) för att säga "skapa det *här*, i mappen jag redan står i".

> **Vad gör `npm create vite@latest . -- --template react-ts`?**
> - `npm create` är en genväg för att köra ett scaffolding-paket
> - `vite@latest` — använd senaste versionen av Vites scaffolding
> - `.` — skapa i nuvarande mapp (istället för en ny undermapp)
> - `-- --template react-ts` — hoppa över den interaktiva menyn, välj React + TypeScript direkt

Du kan få en fråga om package name - om du har namngett din mapp med bara *A-Z, a-z, 0-9, _ och -* så bör Vite räkna ut ett föreslaget package-name utifrån mapp-namnet och tryck då bara ENTER för att bekräfta.

Du får fortfarande en fråga *"Install with npm and start now?"*. Du kan svara ja för att få till npm-installation av nödvändiga paket för Vite+React+TS, men sedan stänga ned servern direkt med *Ctrl+C*.

### Vad fick vi?

```
from-scratch-play/
├── index.html            ← HTML-entryn (Vite serverar denna)
├── package.json          ← Vår enda package.json!
├── vite.config.ts        ← Vite-konfiguration (här lägger vi proxy senare)
├── tsconfig.json         ← TypeScript-config för appen
├── tsconfig.app.json     ← TS-config specifik för app-koden
├── tsconfig.node.json    ← TS-config för Node-sidan (vite.config etc.)
├── eslint.config.js      ← ESLint-konfiguration
├── public/               ← Statiska filer (kopieras rakt av vid build)
│   └── favicon.svg
└── src/                  ← React-källkoden
    ├── App.tsx
    ├── App.css
    ├── main.tsx          ← Entry point — renderar <App /> till DOM
    ├── index.css
    └── vite-env.d.ts     ← TypeScript-deklarationer för Vite
```

Testa att allt fungerar:

```bash
npm install
npm run dev
```

Du bör se Vites välkomstsida på `http://localhost:5173`. Stoppa servern med `Ctrl+C`.

---

## Steg 2: Skapa backend-strukturen med .NET

Nu skapar vi C#-backenden. Vi lägger allt .NET-relaterat i en `backend/`-mapp för att hålla det separerat från frontend-koden.

### Skapa mappen och lösningen

```bash
mkdir backend
cd backend
```

Först skapar vi en **solution** — det är .NET:s sätt att gruppera ihop relaterade projekt:

```bash
dotnet new sln -n MyApp
```

> **`.slnx` — det nya solution-formatet**
>
> Från och med .NET 10 skapar `dotnet new sln` en `.slnx`-fil istället för den äldre `.sln`-filen. Det nya formatet är enklare XML, lättare att läsa och diff:a i git. Funktionen är densamma — det är bara formatet som moderniserats.

### Skapa API-projektet

Vi använder `dotnet new web` — den mest minimala webbmallen. Ingen Swagger, inga exempelroutes, bara en tom app:

```bash
dotnet new web -n App -o App
```

### Skapa testprojektet

```bash
dotnet new xunit -n App.Tests -o App.Tests
```

### Koppla ihop allt

Lägg till båda projekten i lösningen:

```bash
dotnet sln MyApp.slnx add App/App.csproj App.Tests/App.Tests.csproj
```

Lägg till en **project reference** så att testprojektet kan komma åt koden i App-projektet:

```bash
cd App.Tests
dotnet add reference ../App/App.csproj
cd ../..
```

> **Vad är en solution och project references?**
>
> En `.slnx`-fil (solution) är som en "paraply-fil" som grupperar ihop relaterade `.csproj`-projekt. Den säger: "dessa projekt hör ihop."
>
> En **project reference** (`<ProjectReference>` i `.csproj`) säger: "detta projekt *beror på* ett annat projekt." Testprojektet behöver kunna referera till klasserna i App-projektet — därför lägger vi till referensen.
>
> Strukturen blir:
> ```
> backend/
> ├── MyApp.slnx             ← Binder ihop projekten
> ├── App/
> │   ├── App.csproj         ← API-projektet
> │   └── Program.cs         ← Vår Minimal API
> └── App.Tests/
>     ├── App.Tests.csproj   ← Testprojektet (refererar till App)
>     └── UnitTest1.cs       ← Genererad testfil
> ```
>
> De är "systermappar" — separata projekt som lever sida vid sida, bundna av lösningen.

Verifiera att allt kompilerar:

```bash
cd backend
dotnet build MyApp.slnx
cd ..
```

---

## Steg 3: Koppla ihop frontend och backend

Nu ska vi få frontend och backend att prata med varandra — och kunna starta allt med ett enda kommando.

### 3.1 Ge backenden en API-route

Öppna `backend/App/Program.cs`. Just nu ser den ut så här:

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```

Ändra den till:

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/api/hello", () => new { message = "Hello from .NET!" });

app.Run();
```

Vi byter ut `/` mot `/api/hello` och returnerar ett objekt istället för en sträng. Minimal API serialiserar det automatiskt till JSON.

> **Varför `/api/`-prefix?**
>
> Alla våra backend-routes börjar med `/api/`. Det gör att Vite vet exakt vilka anrop som ska proxy:as till backenden och vilka som ska hanteras av frontenden. Enkelt, tydligt, inga konflikter.

### 3.2 Sätt en fast port för backenden

Öppna `backend/App/Properties/launchSettings.json` och ändra porten till `5001` i `http`-profilen:

```json
"applicationUrl": "http://localhost:5001"
```

> Porten som `dotnet new web` genererar är slumpmässig. Vi sätter en fast port så att vår Vite-proxy alltid vet var backenden finns.

### 3.3 Konfigurera Vite proxy

Öppna `vite.config.ts` och lägg till proxy-konfiguration:

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': 'http://localhost:5001'
    }
  }
})
```

**Vad händer nu?**

1. Frontenden kör på `http://localhost:5173` (Vites dev server)
2. Backenden kör på `http://localhost:5001` (.NET)
3. När frontenden gör ett `fetch('/api/hello')` fångar Vite upp det och skickar vidare till `http://localhost:5001/api/hello`
4. Webbläsaren tror att allt kommer från samma server — **ingen CORS behövs**

### 3.4 Installera `concurrently`

Vi behöver kunna starta båda servrarna med ett enda kommando:

```bash
npm install -D concurrently
```

> Vi använder `-D` (dev dependency) eftersom `concurrently` bara behövs under utveckling, inte i produktion.

### 3.5 Uppdatera npm-scripts

Öppna `package.json` och uppdatera `scripts`-sektionen:

```json
"scripts": {
  "start": "concurrently -n backend,vite npm:backend npm:dev",
  "dev": "vite",
  "backend": "dotnet run --project backend/App",
  "build": "tsc -b && vite build",
  "lint": "eslint .",
  "preview": "vite preview"
}
```

| Script | Vad det gör |
|--------|-------------|
| `npm start` | Startar **både** backend och frontend parallellt |
| `npm run dev` | Startar bara Vite (om backenden redan kör) |
| `npm run backend` | Startar bara .NET-backenden |

> **Varför `start` istället för `dev` för bådas start?**
>
> `npm start` är ett av npm:s inbyggda script — man slipper skriva `run`. Smidigt! Och `npm run dev` behåller sin naturliga betydelse: "starta dev-servern" (Vite).
>
> **Vad gör `-n backend,vite`?**
>
> Flaggan namnger processerna. I terminalen syns varje rad prefixad med `[backend]` eller `[vite]` så man ser vad som kommer varifrån.
>
> **Vad är `npm:backend`?**
>
> Det är concurrently:s shorthand för `npm run backend`. Fördelen: inga citattecken behövs — det undviker problem med att olika terminaler (cmd.exe, PowerShell, bash) tolkar escaped quotes olika.

### 3.6 Testa att allt fungerar

```bash
npm start
```

Du bör se något liknande i terminalen:

```
[backend] Now listening on: http://localhost:5001
[vite]   VITE v8.x.x  ready in XXX ms
[vite]   ➜  Local:   http://localhost:5173/
```

Båda servrarna kör! Öppna `http://localhost:5173` i webbläsaren — du ser Vites välkomstsida.

Testa proxy:n genom att öppna `http://localhost:5173/api/hello` i webbläsaren (eller en ny terminal):

```bash
curl http://localhost:5173/api/hello
```

Du bör se:

```json
{"message":"Hello from .NET!"}
```

Anropet gick till Vite (port 5173) som proxy:ade det vidare till .NET (port 5001). Frontenden och backenden är nu ihopkopplade!

Stoppa servrarna med `Ctrl+C`.

### 3.7 Kontrollera att frontend kan göra en fetch från backend-api:t

Byt ut innehållet i **src/App.tsx** till: 


```tsx
import { useState, useEffect } from 'react'


export default function App() {

  const [message,setMessage] = useState('');

  useEffect(()=>{
    (async ()=>{
      const response = await fetch('/api/hello');
      const data = await response.json();
      setMessage((data as any).message);
    })();
  }, []);

  return <>
    <h1>A message from our backend</h1>
    <p>{message}</p>
  </>

}
```

---

## Steg 4: Enhetstester med xUnit

Vi har redan ett testprojekt (`backend/App.Tests/`) kopplat till vår backend via en project reference. Nu lägger vi till något att testa.

### 4.1 Skapa en klass att testa

Skapa filen `backend/App/Greeter.cs`:

```csharp
public class Greeter
{
    public string Greet(string name)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Name cannot be empty");

        return $"Hello, {name}!";
    }
}
```

### 4.2 Använd den i en ny route

Uppdatera `backend/App/Program.cs` — lägg till en route som använder `Greeter`:

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

var greeter = new Greeter();

app.MapGet("/api/hello", () => new { message = "Hello from .NET!" });
app.MapGet("/api/greet/{name}", (string name) => new { message = greeter.Greet(name) });

app.Run();
```

Testa den nya routen:

```bash
npm run backend
# I en annan terminal:
curl http://localhost:5001/api/greet/Anna
# → {"message":"Hello, Anna!"}
```

### 4.3 Skriv testerna

Ersätt innehållet i `backend/App.Tests/UnitTest1.cs` (eller skapa en ny fil `GreeterTest.cs` och ta bort `UnitTest1.cs`):

```csharp
namespace App.Tests;

public class GreeterTest
{
    private readonly Greeter _greeter = new();

    [Fact]
    public void Greet_ReturnsGreetingWithName()
    {
        var result = _greeter.Greet("Anna");
        Assert.Equal("Hello, Anna!", result);
    }

    [Theory]
    [InlineData("Erik", "Hello, Erik!")]
    [InlineData("Maria", "Hello, Maria!")]
    public void Greet_WorksWithDifferentNames(string name, string expected)
    {
        var result = _greeter.Greet(name);
        Assert.Equal(expected, result);
    }

    [Theory]
    [InlineData("")]
    [InlineData("   ")]
    [InlineData(null)]
    public void Greet_ThrowsOnEmptyName(string? name)
    {
        Assert.Throws<ArgumentException>(() => _greeter.Greet(name!));
    }
}
```

> **xUnit-begrepp:**
>
> - **`[Fact]`** — ett enskilt test, körs en gång
> - **`[Theory]` + `[InlineData]`** — samma test körs med olika data (parametriserat test)
> - **`Assert.Equal(expected, actual)`** — kontrollera att värdet stämmer
> - **`Assert.Throws<T>()`** — kontrollera att rätt exception kastas

### 4.4 Lägg till npm-script

Lägg till ett `test:xunit`-script i `package.json`:

```json
"test:xunit": "dotnet test backend/MyApp.slnx"
```

### 4.5 Kör testerna

```bash
npm run test:xunit
```

Du bör se:

```
Passed!  - Failed: 0, Passed: 7, Skipped: 0, Total: 7
```

---

## Steg 5: API-tester med post-they

Nu testar vi våra REST-endpoints utifrån — som en riktig klient skulle göra det. Vi använder **post-they** som låter oss skriva Postman-tester som vanliga JavaScript-filer i VS Code, utan att behöva öppna Postman.

### 5.1 Installera post-they

```bash
npm install -D post-they
```

> **post-they** kompilerar våra testfiler till en Postman collection som sedan körs med **Newman** (Postman:s CLI-runner). Newman ingår som dependency i post-they, så vi behöver inte installera det separat.
>
> **Notera:** Du kommer troligtvis se varningar från `npm audit` efter installationen. Dessa kommer från Newmans interna beroenden (lodash, handlebars, underscore m.fl.) som inte har uppdaterats till säkra versioner. Det påverkar inte säkerheten i ditt projekt — det är testverktyg som bara körs lokalt, aldrig i produktion.

### 5.2 Skapa teststrukturen

Vi skapar mappstrukturen manuellt här eftersom vi bara behöver två enkla requests. 

```bash
mkdir -p api-tests/requests
```

> **Tips:** post-they har ett `init`-kommando (`post-they init api-tests`) som scaffoldar ett komplett CRUD-exempel med looping, testdata och hjälpfunktioner. Kör det gärna i en separat mapp om du vill se ett mer avancerat exempel!

### 5.3 Skapa collection-filen

Skapa `api-tests/collection.js` — den definierar vilka tester som ska köras och i vilken ordning:

```javascript
import getHello from './requests/get-hello.js';
import greetName from './requests/greet-name.js';

export const name = 'FromScratchAPI';

export function preRequest() {
  pm.variables.set('baseUrl', 'http://localhost:5001');
}

export const order = [
  getHello,
  greetName
];
```

> `preRequest()` körs före varje request — här sätter vi `baseUrl` så vi slipper upprepa den i varje testfil.

### 5.4 Skapa testfilerna

Skapa `api-tests/requests/get-hello.js`:

```javascript
export default {
  method: 'GET',
  url: '{{baseUrl}}/api/hello'
};

export function postResponse() {
  pm.test('Status code is 200', () => pm.response.to.have.status(200));

  const json = pm.response.json();
  pm.test('Response has message field', () =>
    pm.expect(json.message).to.equal('Hello from .NET!')
  );
}
```

Skapa `api-tests/requests/greet-name.js`:

```javascript
export default {
  method: 'GET',
  url: '{{baseUrl}}/api/greet/Anna'
};

export function postResponse() {
  pm.test('Status code is 200', () => pm.response.to.have.status(200));

  const json = pm.response.json();
  pm.test('Greeting contains name', () =>
    pm.expect(json.message).to.equal('Hello, Anna!')
  );
}
```

> **post-they-mönstret:**
>
> - Varje request-fil exporterar ett **default-objekt** med `method` och `url`
> - `postResponse()` körs efter att svaret kommit — här skriver vi våra assertions
> - `{{baseUrl}}` ersätts med variabeln vi satte i `preRequest()`
> - **Undvik variabelnamnet `data`** i `postResponse()` — det är reserverat av post-they för looping-funktionalitet

### 5.5 Lägg till npm-script

Lägg till ett `test:api`-script i `package.json`:

```json
"test:api": "post-they run api-tests"
```

### 5.6 Kör API-testerna

Starta backenden i en terminal:

```bash
npm run backend
```

Kör testerna i en annan terminal:

```bash
npm run test:api
```

> **Obs:** API-testerna kräver att backenden kör! De testar mot det riktiga API:et.

Du bör se:

```
→ Get Hello
  GET http://localhost:5001/api/hello [200 OK]
  ✓  Status code is 200
  ✓  Response has message field

→ Greet Name
  GET http://localhost:5001/api/greet/Anna [200 OK]
  ✓  Status code is 200
  ✓  Greeting contains name
```

> **Alternativ utan post-they — Newman direkt med exporterade collections**
>
> Om du föredrar att skriva tester i Postman och exportera som JSON kan du använda Newman direkt:
>
> ```bash
> npm install -D newman
> ```
>
> Lägg den exporterade `.json`-filen i `api-tests/` och peka på den i ditt npm-script:
>
> ```json
> "test:api": "newman run api-tests/my-collection.json"
> ```
>
> Har du flera collections, kedja dem:
>
> ```json
> "test:api": "newman run api-tests/users.json && newman run api-tests/products.json"
> ```
>
> **Obs:** Använd inte glob-mönster (`api-tests/*.json`) — det fungerar inte i Windows cmd.exe. Peka alltid på specifika filer.

---

## Steg 6: E2E-tester med Playwright + BDD

Nu sätter vi upp end-to-end-tester som kör i en riktig webbläsare. Vi använder **Playwright** tillsammans med **playwright-bdd** som låter oss skriva tester i **Gherkin-format** (`.feature`-filer) — ett sätt att beskriva tester på nästan vanlig svenska.

### 6.1 Installera Playwright och BDD-stöd

```bash
npm install -D @playwright/test playwright-bdd
npx playwright install chromium
```

> `npx playwright install chromium` laddar ner en Chromium-webbläsare som Playwright styr. Du behöver bara göra detta en gång.

### 6.2 Skapa mappstrukturen

```bash
mkdir -p e2e/features e2e/steps
```

### 6.3 Konfigurera Playwright

Skapa `playwright.config.js` i projektroten:

```javascript
import { defineConfig, devices } from '@playwright/test';
import { defineBddConfig } from 'playwright-bdd';

const testDir = defineBddConfig({
  features: 'e2e/features/**/*.feature',
  steps: 'e2e/steps/**/*.js',
  outputDir: '.features-gen'
});

export default defineConfig({
  testDir,
  timeout: 30_000,
  expect: {
    timeout: 10_000
  },
  reporter: [
    ['list'],
    ['html', { outputFolder: 'playwright-report', open: 'never' }]
  ],
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'retain-on-failure',
    screenshot: 'only-on-failure'
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    }
  ]
});
```

> **Vad gör `defineBddConfig`?**
>
> Den kopplar ihop `.feature`-filer med step-definitioner och genererar Playwright-testfiler automatiskt (i `.features-gen/`). Playwright kör sedan dessa genererade filer.

### 6.4 Skriv en feature-fil

Skapa `e2e/features/smoke.feature`:

```gherkin
Feature: Smoke-test

    Scenario: Startsidan går att öppna
        Given att jag öppnar startsidan
        Then ska jag se en rubrik på nivå 1 på sidan

    Scenario: API:et svarar via proxyn
        Given att jag öppnar "/api/hello" i webbläsaren
        Then ska jag se texten "Hello from .NET!"
```

> **Gherkin** är ett sätt att beskriva beteende i nästan naturligt språk. Varje `Given`/`When`/`Then`-rad kopplas till en JavaScript-funktion (en "step definition").

**Tips:** För syntax-highlighting och viss linting av Gherkin/feature filer installera VSC-kod tillägget **Cucumber**.

### 6.5 Skriv step-definitioner

Skapa `e2e/steps/smoke.steps.js`:

```javascript
import { expect } from '@playwright/test';
import { createBdd } from 'playwright-bdd';

const { Given, Then } = createBdd();

Given('att jag öppnar startsidan', async ({ page }) => {
  await page.goto('/');
});

Given('att jag öppnar {string} i webbläsaren', async ({ page }, path) => {
  await page.goto(path);
});

Then('ska jag se en rubrik på nivå {int} på sidan', async ({ page }, level) => {
  const heading = page.locator(`h${level}`);
  await expect(heading).toBeVisible();
});

Then('ska jag se texten {string}', async ({ page }, text) => {
  await expect(page.locator('body')).toContainText(text);
});
```

> **Mönstret:**
>
> - `createBdd()` ger oss `Given`, `When`, `Then`-funktioner
> - Varje step tar emot Playwright-fixtures (t.ex. `page`) och eventuella parametrar från feature-filen
> - **`{string}`** matchar text inom citattecken — `"Hello from .NET!"` → `text`
> - **`{int}`** matchar ett heltal — `nivå 1` → `level` blir siffran `1`
> - Vi använder Playwrights inbyggda `expect` för assertions

### 6.6 Lägg till npm-script

Lägg till ett `test:e2e`-script i `package.json`:

```json
"test:e2e": "npx bddgen && npx playwright test"
```

> `bddgen` genererar Playwright-testfiler från `.feature`-filerna. Det måste köras innan `playwright test`.

### 6.7 Kör E2E-testerna

E2E-testerna kräver att **både backend och frontend kör** (de testar i webbläsaren via Vite-proxyn):

```bash
# Terminal 1:
npm start

# Terminal 2:
npm run test:e2e
```

Du bör se:

```
  ✓  [chromium] › Smoke-test › Startsidan går att öppna
  ✓  [chromium] › Smoke-test › API:et svarar via proxyn

  2 passed
```

---

## Steg 7: npm-scripts — kör alla tester

Nu har vi alla testtyper på plats. Här är hela `scripts`-sektionen i `package.json`:

```json
"scripts": {
  "start": "concurrently -n backend,vite npm:backend npm:dev",
  "dev": "vite",
  "backend": "dotnet run --project backend/App",
  "test": "npm run test:xunit && npm run test:api && npm run test:vitest && npm run test:e2e",
  "test:xunit": "dotnet test backend/MyApp.slnx",
  "test:api": "post-they run api-tests",
  "test:e2e": "npx bddgen && npx playwright test",
  "test:vitest": "vitest run",
  "build": "tsc -b && vite build",
  "lint": "eslint .",
  "preview": "vite preview"
}
```

| Script | Vad det kör | Kräver att servrarna kör? |
|--------|-------------|--------------------------|
| `npm run test:xunit` | xUnit enhetstester | Nej |
| `npm run test:api` | post-they API-tester | Ja — backenden |
| `npm run test:e2e` | Playwright BDD-tester | Ja — backend + frontend |
| `npm run test:vitest` | ViTest komponenttester | Nej |
| `npm test` | Alla ovanstående i sekvens | Ja |

> **Obs:** `npm run test:xunit` och `npm run test:vitest` kan köras utan att starta servrarna — de är rena enhetstester. API- och E2E-tester kräver däremot att backend (och för E2E även frontend) kör. Starta dem med `npm start` i en separat terminal innan du kör `npm test`.

---

## Steg 8: Komponenttester med ViTest

Nu testar vi React-komponenter. Problemet: våra komponenter renderar till en DOM — men Node.js har ingen DOM. Vi behöver **jsdom**, ett bibliotek som simulerar en webbläsarens DOM i Node.

### 8.1 Installera ViTest och testbibliotek

```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom
```

| Paket | Vad det gör |
|-------|-------------|
| `vitest` | Testramverket (snabbt, Vite-native) |
| `@testing-library/react` | `render()`, `screen`, `fireEvent` — verktyg för att rendera och interagera med komponenter |
| `@testing-library/jest-dom` | Extra assertions som `toBeInTheDocument()` |
| `jsdom` | Simulerar en webbläsar-DOM i Node.js |

### 8.2 Konfigurera ViTest

Skapa `vitest.config.ts` i projektroten:

```typescript
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
    include: ['src/**/*.test.{ts,tsx}']
  }
})
```

> **Varför `environment: 'jsdom'`?**
>
> React-komponenter använder `document.createElement()`, `window`, och andra webbläsar-API:er. Node.js har inget av detta. Med `jsdom` får ViTest en simulerad DOM som gör att `render(<App />)` faktiskt fungerar.
>
> **Varför `include`?**
>
> Vi begränsar ViTest till `src/`-mappen. Annars plockar den upp Playwrights genererade testfiler i `.features-gen/` och kraschar.

### 8.3 Skapa setup-fil

Skapa `src/test/setup.ts`:

```typescript
import { cleanup } from '@testing-library/react'
import '@testing-library/jest-dom/vitest'
import { afterEach } from 'vitest'

afterEach(() => {
  cleanup()
})
```

> `cleanup()` rensar DOM:en efter varje test så att tester inte påverkar varandra. Importen av `@testing-library/jest-dom/vitest` ger oss assertions som `toBeInTheDocument()`.

### 8.4 Skriv ett komponenttest

Skapa `src/test/App.test.tsx`:

```tsx
import { render, screen } from '@testing-library/react'
import { describe, it, expect, vi, beforeEach } from 'vitest'
import App from '../App'

describe('App', () => {
  beforeEach(() => {
    // Mocka fetch — det finns ingen riktig backend under testning
    vi.spyOn(globalThis, 'fetch').mockResolvedValue({
      json: async () => ({ message: 'Hello from mock!' })
    } as Response)
  })

  it('visar en rubrik', () => {
    render(<App />)
    expect(screen.getByText('A message from our backend')).toBeInTheDocument()
  })

  it('visar meddelandet från API:et', async () => {
    render(<App />)
    expect(await screen.findByText('Hello from mock!')).toBeInTheDocument()
  })
})
```

> **Varför mockar vi `fetch`?**
>
> Vår `App`-komponent gör `fetch('/api/hello')` i en `useEffect`. Men under testning finns ingen backend — det är därför det är ett *enhetstest*, inte ett integrationstest. Vi använder `vi.spyOn(global, 'fetch')` för att ersätta den riktiga `fetch` med en mockad version som returnerar kontrollerad data.
>
> - `screen.getByText()` — hittar ett element via dess text (kastar fel om det inte finns)
> - `screen.findByText()` — som `getByText` men väntar asynkront (behövs för data som laddas via `useEffect`)

### 8.5 Lägg till npm-script

Lägg till ett `test:vitest`-script i `package.json`:

```json
"test:vitest": "vitest run"
```

### 8.6 Kör testerna

```bash
npm run test:vitest
```

Du bör se:

```
 ✓ src/test/App.test.tsx (2 tests) 2 passed
 
 Test Files  1 passed (1)
      Tests  2 passed (2)
```

> **Tips:** Under utveckling kan du köra `npx vitest` (utan `run`) för watch mode — testerna körs om automatiskt när du sparar en fil.

---

## Reflektion: Playwright eller ViTest — när använder man vad?

Både Playwright och ViTest kan testa att "användaren ser rätt sak på skärmen". Så varför ha båda?

**ViTest + jsdom** kör i Node.js med en *simulerad* DOM. Det är snabbt (millisekunder) men du testar mot en förenkling — jsdom stödjer inte CSS-layout, riktiga nätverksanrop eller webbläsar-specifikt beteende. Allt som komponenten beror på (fetch, router, context) måste du mocka själv.

**Playwright** kör i en *riktig webbläsare* mot hela stacken — frontend, backend, databas. Inget behöver mockas, men varje test tar sekunder istället för millisekunder, och du behöver ha servrarna igång.

| | ViTest + jsdom | Playwright |
|---|---|---|
| **Hastighet** | Mycket snabbt (~ms) | Långsammare (~sekunder) |
| **DOM** | Simulerad (jsdom) | Riktig webbläsare |
| **Backend** | Mockad | Riktig |
| **Mockning** | Du måste mocka fetch, router etc. | Inget behöver mockas |
| **Testar** | En komponent isolerat | Hela flödet som användaren upplever det |
| **Hittar** | Logikfel i komponenter | Integrationsproblem mellan frontend, backend, databas |

**Ju mer komplicerad webbläsarlogik din komponent använder, desto mer måste du mocka i ViTest.** I vårt enkla exempel behövde vi bara mocka `fetch`. Men en komponent som använder `localStorage`, `IntersectionObserver`, `canvas`, `navigator.geolocation`, drag-and-drop eller animationer kräver allt fler mock-bibliotek och simuleringar — och då börjar man testa sin mock-setup lika mycket som sin komponent. Med Playwright slipper man det helt — allt finns i webbläsaren på riktigt.

**Tumregel:** Använd ViTest för att snabbt verifiera att enskilda komponenter beter sig rätt — speciellt de med enkel webbläsarinteraktion. Använd Playwright för att verifiera att allt fungerar ihop, och för komponenter med komplex webbläsarlogik där mockningsarbetet inte är värt besväret.

De *överlappar* delvis — och det är okej. Det är bättre att ha två tester som fångar samma bugg från olika håll än att ha en blind fläck som ingen testar alls.