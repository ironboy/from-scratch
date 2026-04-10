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
npm run dev
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
  "start": "concurrently -n backend,vite \"npm run backend\" \"npm run dev\"",
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
