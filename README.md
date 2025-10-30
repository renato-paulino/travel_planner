AI TRAVEL PLANNER — AZURE-FIRST FULL STACK PROJECT
=================================================

OVERVIEW
---------
AI Travel Planner is a full-stack, Azure-first project that aggregates flight, hotel, and weather data,
and uses AI to recommend the best time to travel to any destination.

You will:
- Practice ASP.NET Core backend development and architecture
- Work with Azure App Service, Azure Functions, Static Web Apps
- Integrate external APIs (flights, hotels, weather)
- Add AI functionality via Hugging Face or Ollama
- Implement CI/CD with GitHub Actions
- Write tests using xUnit and Moq
- Stay entirely within free-tier services

--------------------------------------------------
TECH STACK
--------------------------------------------------
Backend: ASP.NET Core 8 (Azure App Service Free F1)
Database: PostgreSQL (Neon.tech Free)
AI Layer: Hugging Face API / Ollama (Free)
Frontend: React (Vite + TypeScript) or Blazor (Azure Static Web Apps Free)
Automation: Azure Functions (Free 1M exec/month)
Monitoring: Application Insights (Free 5GB/month)
Storage: Azure Blob Storage (Free 5GB/12mo)
CI/CD: GitHub Actions → Azure (Free)

--------------------------------------------------
ARCHITECTURE
--------------------------------------------------
Frontend (React or Blazor on Azure Static Web Apps)
     |
     v
ASP.NET Core Web API (Azure App Service)
  - TravelService
  - Flight / Hotel / Weather API clients
  - AI (Hugging Face or Ollama)
  - In-memory caching
     |
     v
Neon PostgreSQL Database (Free)
     |
     +-- Azure Function (prefetch scheduler)
     +-- Application Insights (monitoring and logs)

--------------------------------------------------
STEP 0 — PROJECT SETUP
--------------------------------------------------
1. Create GitHub repository: travel-planner
2. Branches:
   - main (production)
   - dev (development)
3. Folder structure:
   travel-planner/
       API/
       frontend/
       .github/workflows/
       infra/
       README.md

--------------------------------------------------
STEP 1 — BACKEND SKELETON + TESTS (xUnit + Moq)
--------------------------------------------------
Goal: Run an ASP.NET Core API locally with Swagger and a /health endpoint, plus working unit tests.

Commands:
    dotnet new sln -n TravelPlanner
    dotnet new webapi -n TravelPlanner.API -o API/TravelPlanner.API --framework net8.0
    dotnet new classlib -n TravelPlanner.Application -o API/TravelPlanner.Application --framework net8.0
    dotnet new classlib -n TravelPlanner.Domain -o API/TravelPlanner.Domain --framework net8.0
    dotnet new classlib -n TravelPlanner.Infrastructure -o API/TravelPlanner.Infrastructure --framework net8.0
    dotnet new xunit -n TravelPlanner.Tests -o API/TravelPlanner.Tests --framework net8.0
    dotnet sln API/TravelPlanner.sln add API/**/**.csproj

Project references:
    dotnet add API/TravelPlanner.API/TravelPlanner.API.csproj reference API/TravelPlanner.Application/TravelPlanner.Application.csproj API/TravelPlanner.Infrastructure/TravelPlanner.Infrastructure.csproj API/TravelPlanner.Domain/TravelPlanner.Domain.csproj
    dotnet add API/TravelPlanner.Application/TravelPlanner.Application.csproj reference API/TravelPlanner.Domain/TravelPlanner.Domain.csproj
    dotnet add API/TravelPlanner.Infrastructure/TravelPlanner.Infrastructure.csproj reference API/TravelPlanner.Application/TravelPlanner.Application.csproj API/TravelPlanner.Domain/TravelPlanner.Domain.csproj
    dotnet add API/TravelPlanner.Tests/TravelPlanner.Tests.csproj reference API/TravelPlanner.API/TravelPlanner.API.csproj

    ✅ Benefits of This Structure
    🔒 Dependency Rule: Inner layers never depend on outer layers
    🔄 Testability: Easy to mock external dependencies
    🛡️ Separation of Concerns: Each layer has a single responsibility
    🔧 Maintainability: Changes in outer layers don't affect inner ones
    📦 Domain Independence: Core business logic is framework-agnostic

    TravelPlanner.API references:
    ├── Application ← Business logic and use cases
    ├── Infrastructure ← Database and external API implementations  
    └── Domain ← Core entities for DTOs and validation

    TravelPlanner.Application references:
    └── Domain ← Needs entities and business rules

    TravelPlanner.Infrastructure references:
    ├── Application ← Implements interfaces defined here
    └── Domain ← Needs entities for data mapping

    TravelPlanner.Tests references:
    └── API ← Tests the complete application through HTTP endpoints

    This is a classic implementation of Clean Architecture (also known as Onion Architecture), where the Domain is at the center and dependencies flow inward, ensuring your business logic remains pure and testable.

NuGet packages:
    dotnet add API/TravelPlanner.API package Swashbuckle.AspNetCore
    dotnet add API/TravelPlanner.API package Serilog.AspNetCore
    dotnet add API/TravelPlanner.Tests package Moq
    dotnet add API/TravelPlanner.Tests package FluentAssertions
    dotnet add API/TravelPlanner.Tests package Microsoft.AspNetCore.Mvc.Testing

Minimal API Example (Program.cs):
---------------------------------
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
var app = builder.Build();
app.MapGet("/health", () => Results.Ok("Healthy"));
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
app.Run();
public partial class Program { } // required for testing

xUnit Test Example:
-------------------
using System.Net;
using FluentAssertions;
using Microsoft.AspNetCore.Mvc.Testing;
using Xunit;

namespace TravelPlanner.Tests;
public class HealthEndpointTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    public HealthEndpointTests(WebApplicationFactory<Program> factory) => _factory = factory;

    [Fact]
    public async Task Health_ReturnsOk()
    {
        var client = _factory.CreateClient();
        var response = await client.GetAsync("/health");
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        (await response.Content.ReadAsStringAsync()).Should().Contain("Healthy");
    }
}

Validation commands:
--------------------
    dotnet restore
    dotnet build
    dotnet test API/TravelPlanner.Tests
    dotnet run --project API/TravelPlanner.API

Documentation:
--------------
Minimal APIs: https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis
xUnit Testing: https://learn.microsoft.com/dotnet/core/testing/unit-testing-csharp-with-xunit
Moq Quickstart: https://github.com/moq/moq4/wiki/Quickstart

--------------------------------------------------
STEP 2 — DATABASE (NEON + EF CORE)
--------------------------------------------------
Goal: Add persistence for searches and cached results.

Commands:
    dotnet add API/TravelPlanner.Infrastructure package Npgsql.EntityFrameworkCore.PostgreSQL
    dotnet add API/TravelPlanner.API package Microsoft.EntityFrameworkCore.Design
    dotnet tool install --global dotnet-ef

Setup Secrets:
    dotnet user-secrets init --project API/TravelPlanner.API
    dotnet user-secrets set "ConnectionStrings:Default" "<neon-connection-string>" --project API/TravelPlanner.API

Example DbContext:
------------------
public class AppDbContext : DbContext
{
    public DbSet<Search> Searches => Set<Search>();
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
}

Documentation:
--------------
EF Core: https://learn.microsoft.com/ef/core/
Npgsql Provider: https://learn.microsoft.com/ef/core/providers/npgsql/

--------------------------------------------------
STEP 3 — EXTERNAL APIS
--------------------------------------------------
Goal: Fetch live data for weather, flights, and hotels.

APIs:
Weather: OpenWeatherMap (https://openweathermap.org/api)
Flights: AviationStack (https://aviationstack.com)
Hotels: HotelAPI via RapidAPI (https://rapidapi.com)

Store keys:
    dotnet user-secrets set "ApiKeys:OpenWeather" "<key>"
    dotnet user-secrets set "ApiKeys:Flights" "<key>"
    dotnet user-secrets set "ApiKeys:Hotels" "<key>"

--------------------------------------------------
STEP 4 — AI LAYER (HUGGING FACE / OLLAMA)
--------------------------------------------------
Goal: Generate AI summaries and travel recommendations.

Configuration:
    "AI": {
        "Provider": "HuggingFace",
        "HuggingFaceApiKey": "<key>"
    }

Interface:
----------
public interface IAiService
{
    Task<string> SummarizeAsync(TravelAggregate data, UserPreferences prefs);
}

Documentation:
--------------
Hugging Face API: https://huggingface.co/inference-api
Ollama: https://ollama.com

--------------------------------------------------
STEP 5 — FRONTEND (REACT + VITE + TS)
--------------------------------------------------
Goal: Build a simple frontend hosted on Azure Static Web Apps.

Commands:
    npm create vite@latest frontend -- --template react-ts
    cd frontend
    npm i axios recharts
    npm run dev

Pages:
    /search   - search form
    /results  - flights, hotels, weather, AI summary

Documentation:
--------------
Azure Static Web Apps: https://learn.microsoft.com/azure/static-web-apps/overview

--------------------------------------------------
STEP 6 — AZURE RESOURCES
--------------------------------------------------
Create:
    Resource Group: rg-travel-planner
    App Service Plan (Free F1)
    Web App: travel-planner-api
    Static Web App: frontend
    Application Insights
    (Optional) Blob Storage

Configuration (App Service):
    ConnectionStrings__Default = <neon>
    ApiKeys__OpenWeather = <key>
    ApiKeys__Flights = <key>
    ApiKeys__Hotels = <key>
    AI__Provider = HuggingFace
    AI__HuggingFaceApiKey = <key>

Documentation:
--------------
Deploy ASP.NET Core to Azure App Service:
https://learn.microsoft.com/aspnet/core/host-and-deploy/azure-apps

--------------------------------------------------
STEP 7 — CI/CD (GITHUB ACTIONS)
--------------------------------------------------
Secrets:
    AZURE_WEBAPP_PUBLISH_PROFILE
    AZURE_STATIC_WEB_APPS_API_TOKEN

Workflows:
API Deployment YAML:
--------------------
name: Deploy API to Azure
on:
  push:
    branches: [ "main" ]
    paths: [ "API/**" ]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - run: dotnet restore API/TravelPlanner.sln
      - run: dotnet build API/TravelPlanner.sln --configuration Release
      - run: dotnet publish API/TravelPlanner.API -c Release -o ./publish
      - uses: azure/webapps-deploy@v3
        with:
          app-name: "travel-planner-api"
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          package: ./publish

Frontend Deployment YAML:
-------------------------
name: Deploy Frontend to Azure Static Web Apps
on:
  push:
    branches: [ "main" ]
    paths: [ "frontend/**" ]
jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci --prefix frontend
      - run: npm run build --prefix frontend
      - uses: Azure/static-web-apps-deploy@v2
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
          app_location: "frontend"
          output_location: "dist"

--------------------------------------------------
STEP 8 — AZURE FUNCTION (PREFETCH)
--------------------------------------------------
Goal: Prefetch data daily for performance.

Setup:
    Azure Function (Consumption plan)
    Timer trigger CRON: 0 0 2 * * *
    Logic: prefetch popular destinations

Documentation:
--------------
Azure Functions Timer Trigger:
https://learn.microsoft.com/azure/azure-functions/functions-bindings-timer

--------------------------------------------------
STEP 9 — MONITORING AND HARDENING
--------------------------------------------------
- Enable Application Insights
- Add Serilog logging
- Implement ASP.NET Core Rate Limiting
- Validate input and add global error handling
- Add caching headers to frontend assets

--------------------------------------------------
STEP 10 — POLISH AND PORTFOLIO
--------------------------------------------------
- Add README badges, screenshots, and diagrams
- Record demo GIF
- Add simple API key auth (optional)
- Deploy public demo link (Azure SWA)

--------------------------------------------------
TESTING STRATEGY
--------------------------------------------------
Framework: xUnit + Moq
Unit tests: Services and repositories
Integration tests: WebApplicationFactory in-memory API
Data tests: EF InMemory or Neon test DB

Run tests:
    dotnet test API/TravelPlanner.Tests

--------------------------------------------------
DOCUMENTATION LINKS
--------------------------------------------------
Minimal APIs: https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis
xUnit Testing: https://learn.microsoft.com/dotnet/core/testing/unit-testing-csharp-with-xunit
Moq: https://github.com/moq/moq4/wiki/Quickstart
EF Core PostgreSQL: https://learn.microsoft.com/ef/core/providers/npgsql/
Azure App Service: https://learn.microsoft.com/aspnet/core/host-and-deploy/azure-apps
Static Web Apps: https://learn.microsoft.com/azure/static-web-apps/overview
Azure Functions: https://learn.microsoft.com/azure/azure-functions/functions-bindings-timer
GitHub Actions for Azure: https://learn.microsoft.com/azure/developer/github/github-actions

--------------------------------------------------
QUICK START
--------------------------------------------------
Backend:
    dotnet restore
    dotnet build
    dotnet test API/TravelPlanner.Tests
    dotnet run --project API/TravelPlanner.API

Frontend:
    cd frontend
    npm install
    npm run dev

Local URLs:
    API Swagger: http://localhost:5000/swagger
    Health: http://localhost:5000/health

--------------------------------------------------
LICENSE
--------------------------------------------------
MIT License © 2025
