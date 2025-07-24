# Construindo-um-Sistema-de-Agendamento-de-Tarefas-com-Entity-Framework
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Banco de dados em memória
builder.Services.AddDbContext<TarefaDbContext>(options =>
    options.UseInMemoryDatabase("TarefasDb"));

var app = builder.Build();

// Criar tarefa
app.MapPost("/tarefas", (Tarefa tarefa, TarefaDbContext db) =>
{
    db.Tarefas.Add(tarefa);
    db.SaveChanges();
    return Results.Created($"/tarefas/{tarefa.Id}", tarefa);
});

// Listar todas
app.MapGet("/tarefas", (TarefaDbContext db) =>
    db.Tarefas.ToList());

// Buscar por ID
app.MapGet("/tarefas/{id}", (int id, TarefaDbContext db) =>
    db.Tarefas.Find(id) is Tarefa tarefa ? Results.Ok(tarefa) : Results.NotFound());

// Atualizar tarefa
app.MapPut("/tarefas/{id}", (int id, Tarefa input, TarefaDbContext db) =>
{
    var tarefa = db.Tarefas.Find(id);
    if (tarefa is null) return Results.NotFound();

    tarefa.Descricao = input.Descricao;
    tarefa.Concluida = input.Concluida;
    db.SaveChanges();

    return Results.Ok(tarefa);
});

// Deletar tarefa
app.MapDelete("/tarefas/{id}", (int id, TarefaDbContext db) =>
{
    var tarefa = db.Tarefas.Find(id);
    if (tarefa is null) return Results.NotFound();

    db.Tarefas.Remove(tarefa);
    db.SaveChanges();

    return Results.NoContent();
});

app.Run();

// Modelo
public class Tarefa
{
    public int Id { get; set; }
    public string Descricao { get; set; } = "";
    public bool Concluida { get; set; } = false;
}

// DbContext
public class TarefaDbContext : DbContext
{
    public TarefaDbContext(DbContextOptions options) : base(options) {}
    public DbSet<Tarefa> Tarefas => Set<Tarefa>();
}
using Xunit;
using System.Net;
using System.Net.Http.Headers;
using System.Text;
using Microsoft.AspNetCore.Mvc.Testing;
using System.Threading.Tasks;

public class TarefaApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public TarefaApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task DeveCadastrarTarefa()
    {
        var json = "{\"descricao\":\"Estudar testes\",\"concluida\":false}";
        var response = await _client.PostAsync("/tarefas", new StringContent(json, Encoding.UTF8, "application/json"));

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    }

    [Fact]
    public async Task DeveListarTarefas()
    {
        var response = await _client.GetAsync("/tarefas");
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
}
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "TarefaApi.dll"]
name: Tarefa API Build & Test

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: 7.0.x
    - name: Restore
      run: dotnet restore
    - name: Build
      run: dotnet build --configuration Release
    - name: Test
      run: dotnet test --no-build
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Conexão PostgreSQL
builder.Services.AddDbContext<TarefaDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("PostgresConnection")));

var app = builder.Build();

app.MapPost("/tarefas", (Tarefa tarefa, TarefaDbContext db) =>
{
    db.Tarefas.Add(tarefa);
    db.SaveChanges();
    return Results.Created($"/tarefas/{tarefa.Id}", tarefa);
});

app.MapGet("/tarefas", (TarefaDbContext db) =>
    db.Tarefas.ToList());

app.MapGet("/tarefas/{id}", (int id, TarefaDbContext db) =>
    db.Tarefas.Find(id) is Tarefa tarefa ? Results.Ok(tarefa) : Results.NotFound());

app.MapPut("/tarefas/{id}", (int id, Tarefa input, TarefaDbContext db) =>
{
    var tarefa = db.Tarefas.Find(id);
    if (tarefa is null) return Results.NotFound();

    tarefa.Descricao = input.Descricao;
    tarefa.Concluida = input.Concluida;
    db.SaveChanges();

    return Results.Ok(tarefa);
});

app.MapDelete("/tarefas/{id}", (int id, TarefaDbContext db) =>
{
    var tarefa = db.Tarefas.Find(id);
    if (tarefa is null) return Results.NotFound();

    db.Tarefas.Remove(tarefa);
    db.SaveChanges();

    return Results.NoContent();
});

app.Run();

public class Tarefa
{
    public int Id { get; set; }
    public string Descricao { get; set; } = "";
    public bool Concluida { get; set; } = false;
}

public class TarefaDbContext : DbContext
{
    public TarefaDbContext(DbContextOptions options) : base(options) {}
    public DbSet<Tarefa> Tarefas => Set<Tarefa>();
}
{
  "ConnectionStrings": {
    "PostgresConnection": "Host=localhost;Port=5432;Database=TarefasDb;Username=usuario;Password=senha"
  }
}
