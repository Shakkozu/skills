---
name: Nabrdalik TDD (.NET)
description: Design, write, review, and repair .NET tests using Jakub Nabrdalik's module-focused TDD with NUnit and Reqnroll. Use when choosing test levels, replacing brittle unit or end-to-end tests, writing module or acceptance specs, building sample data and test DSLs, or fixing a slow/flaky .NET test suite.
---

# Nabrdalik TDD — .NET (NUnit + Reqnroll)

> Source: Jakub Nabrdalik, "Improving your Test Driven Development".
> .NET adaptation of the `nabrdalik-shortened` skill. All examples are C#.

**Core idea:** test modules as black boxes through their public API, in milliseconds; add a few real-I/O acceptance tests for critical paths; make every test read like a requirement.

Use **NUnit** for code specs. Use **Reqnroll** when the team genuinely wants business-readable Gherkin. Do not mix levels or duplicate the same coverage in both.

## 1. Test the right level

| Failure | Symptom | Fix |
| --- | --- | --- |
| **Too low** | Per-class tests with mocks; refactors break unrelated tests; 100% coverage still misses production bugs | Test module behavior through its facade |
| **Too high** | Whole system through HTTP/UI; slow `WebApplicationFactory` + DB + broker per case; suites take tens of minutes | Push corner cases down to module specs |

A **module** is a vertical slice / bounded context: owns its data, exposes a facade, hides its adapters, and is a likely microservice candidate. In .NET it can be a project or a folder with `public` facade types and `internal` adapters.

| Level | .NET scope | I/O | Speed | Count |
| --- | --- | --- | --- | --- |
| Module spec | NUnit + plain `new ModuleConfiguration().Facade()` | None / in-memory adapters | Milliseconds | Many |
| Acceptance spec | NUnit + `WebApplicationFactory` / `HttpClient` / Testcontainers | Real core I/O | Seconds | Few |
| Contract spec | Separate project + WireMock.Net / Testcontainers | Real or stubbed external | Minutes | Outside main build |
| Reqnroll spec | Gherkin feature + step definitions | Depends on steps | Depends | Only where valuable |

Pure algorithms may be tested directly. Everything else is module behavior first.

## 2. Rules

1. Test **behavior**, not implementation.
2. Go through the **public API** — facade or HTTP endpoint.
3. **Do not let I/O out.** Wire in-memory adapters inside the module configuration; never add a test-only overload that accepts `IFilmRepository`.
4. **Mock only other modules**, never the module's own repository/DAO/`DbContext`.
5. Module specs use **no `WebApplicationFactory`, `IServiceProvider`, EF Core, `HttpClient`, filesystem, or `Thread.Sleep`**.
6. Use **sample data**: one valid default per DTO, override only what matters.
7. Extract **one obvious way** to interact with the API.
8. Write tests as **specifications**: Given/When/Then in domain language.
9. **Show, don't tell** — if you would draw it, express it in C#.
10. **Remove before adding**; delete incidental detail.
11. Keep module specs in **milliseconds**, acceptance tests in **seconds**.
12. Tests == specifications == requirements.

## 3. Design before writing

For each behavior, fill a row:

| Given | When | Then | Level | I/O? | Sample data / DSL |
| --- | --- | --- | --- | --- | --- |
| a film is in the catalogue | it is requested by title | that film is returned | Module | No | `SampleFilm()` |
| no film has that title | it is requested | not-found error | Module | No | `SampleFilm() with { Title = "x" }` |
| two films exist | all films are requested over HTTP | both appear in JSON | Acceptance | Yes | `FilmApi.GetFilms()` |

Then define:

- **samples** — `SampleFilm()`, `SampleArticle()`
- **named actions** — `PostNewArticle(...)`, `Preview(...)`
- **DSL** — `A(B(), C(E(), F()), D())`, `Add(G()).Under(C()).At(1)`
- **custom assertions** — `Assert.That(updated, Is.EqualTo(Expected(...)))`

If the vocabulary is hard to design, the requirement is not understood yet.

## 4. Module spec pattern (NUnit)

Composition root for tests and local runs:

```csharp
public sealed class FilmConfiguration
{
    public FilmFacade FilmFacade() =>
        new(new InMemoryFilmRepository(), new FilmCreator());
}
```

The application's DI registration uses the real adapter instead:

```csharp
services.AddScoped<IFilmRepository, EfFilmRepository>();
services.AddScoped<FilmFacade>();
```

Do not create a third path for tests that lets them inject a mock repository.

```csharp
[TestFixture]
public sealed class FilmSpec
{
    private readonly FilmFacade _facade = new FilmConfiguration().FilmFacade();

    [Test]
    public void Should_return_a_film_by_title()
    {
        // given
        var film = SampleFilms.SampleFilm();
        _facade.Add(film);

        // when
        var found = _facade.Show(film.Title);

        // then
        Assert.That(found, Is.EqualTo(film));
    }

    [Test]
    public void Should_report_not_found_for_an_unknown_title()
    {
        Assert.Throws<FilmNotFoundException>(() => _facade.Show("missing"));
    }
}
```

No `WebApplicationFactory`. No `IServiceProvider`. No database. Add corner cases freely.

When the module collaborates with **other modules**, mock their public interfaces:

```csharp
var search = Substitute.For<ISearchFacade>();
var promoted = Substitute.For<IPromotedArticleFacade>();
var facade = new ArticleConfiguration().ArticleFacade(search, promoted);
```

`NSubstitute` or `Moq` is fine here. Mocking `IFilmRepository` inside the same module is not.

NUnit essentials:

```csharp
[TestFixture]                  // test class
[Test]                         // test method
[TestCase(1, 2, 3)]            // data-driven case
[SetUp] / [TearDown]           // per-test fixture
[Category("Integration")]      // suite filtering
Assert.That(actual, Is.EqualTo(expected));
Assert.Throws<T>(() => ...);
await Assert.ThrowsAsync<T>(async () => ...);
```

## 5. Sample data, not setup explosions

```csharp
public static class SampleFilms
{
    public static FilmDto SampleFilm() => new(
        Title: "50 shades of Trumpet",
        Type: FilmTypeDto.New);
}

// var noTitle = SampleFilms.SampleFilm() with { Title = null! };
// var oldFilm = SampleFilms.SampleFilm() with { Type = FilmTypeDto.Old };
```

For larger DTOs, keep a valid default and use `with`:

```csharp
public static NewArticleDto SampleNewArticle() => new()
{
    SpaceId = "default",
    Title = "The Juliette Society review",
    Lead = "In her debut erotic novel...",
    PublishDate = new DateTimeOffset(2011, 12, 3, 10, 15, 30, TimeSpan.Zero)
};

// var noTitle   = SampleNewArticle() with { Title = null! };
// var inGuides  = SampleNewArticle() with { SpaceId = "guides" };
```

Rules:

- Default sample is **valid**; override only the field under test.
- Prefer records + `with`; use a builder/object mother for mutable classes.
- Keep samples **deterministic**; inject `TimeProvider` instead of using `DateTime.Now`.
- Reuse the same sample data for integration JSON.
- If nested setup grows, add a named helper — not another 30-line builder chain.
- Sample factories live in test support, not production.

## 6. Integration specs (NUnit + WebApplicationFactory)

Keep real I/O at the edge. Extract every repeated interaction behind a meaningful name.

```csharp
public sealed class FilmApi
{
    private readonly HttpClient _client;
    public FilmApi(HttpClient client) => _client = client;

    public Task<HttpResponseMessage> GetFilms() =>
        _client.GetAsync("/films");

    public Task<HttpResponseMessage> GetFilm(string title) =>
        _client.GetAsync($"/film/{Uri.EscapeDataString(title)}");
}
```

```csharp
[TestFixture]
public sealed class FilmAcceptanceTests
{
    private WebApplicationFactory<Program> _factory = null!;
    private FilmApi _api = null!;
    private FilmFacade _facade = null!;

    [SetUp]
    public void SetUp()
    {
        _factory = new WebApplicationFactory<Program>();
        _api = new FilmApi(_factory.CreateClient());
        _facade = _factory.Services.GetRequiredService<FilmFacade>();
    }

    [TearDown]
    public void TearDown() => _factory.Dispose();

    [Test]
    public async Task Should_expose_films_over_HTTP()
    {
        // given
        var a = SampleFilms.SampleFilm() with { Title = "A" };
        var b = SampleFilms.SampleFilm() with { Title = "B" };
        _facade.Add(a);
        _facade.Add(b);

        // when
        var response = await _api.GetFilms();

        // then
        Assert.That(response.StatusCode, Is.EqualTo(HttpStatusCode.OK));
        var page = await response.Content.ReadFromJsonAsync<FilmPageDto>();
        Assert.That(page!.Content.Select(x => x.Title),
            Is.EquivalentTo(new[] { a.Title, b.Title }));
    }
}
```

For `WebApplicationFactory<Program>` with top-level statements, add this to `Program.cs`:

```csharp
public partial class Program { }
```

Developers should not re-decide endpoint, method, payload, serialization, or sync/async. Answer once; name methods after **domain actions** (`PostNewArticle`, `Preview`, `Publish`), not HTTP verbs.

Assert observable results: DTOs, events, status/body, state read back through the API. Never assert `Verify(repo => repo.Save(...))`, SQL, `DbContext` state, private fields, or logs.

Keep acceptance tests few: main happy path, integration contract, money path. Push corner cases down. Use Testcontainers for a real database/broker only when it is central to the behavior; otherwise use the in-memory adapter or a boundary stub.

## 7. Reqnroll specs

Use Reqnroll (the SpecFlow successor) when the Gherkin is genuinely valuable to the team. Do not wrap every NUnit test in a feature file.

```gherkin
Feature: Film catalogue

Scenario: Return a film by title
  Given a film is in the catalogue
  When the film is requested by title
  Then the system returns that film
```

```csharp
using NUnit.Framework;
using Reqnroll;

[Binding]
public sealed class FilmSteps
{
    private readonly FilmFacade _facade = new FilmConfiguration().FilmFacade();
    private FilmDto _film = null!;
    private FilmDto _found = null!;

    [Given("a film is in the catalogue")]
    public void GivenAFilmIsInTheCatalogue()
    {
        _film = SampleFilms.SampleFilm();
        _facade.Add(_film);
    }

    [When("the film is requested by title")]
    public void WhenTheFilmIsRequestedByTitle()
    {
        _found = _facade.Show(_film.Title);
    }

    [Then("the system returns that film")]
    public void ThenTheSystemReturnsThatFilm()
    {
        Assert.That(_found, Is.EqualTo(_film));
    }
}
```

Packages:

```sh
dotnet add package NUnit
dotnet add package NUnit3TestAdapter
dotnet add package Microsoft.NET.Test.Sdk
dotnet add package Reqnroll.NUnit
```

Rules:

- Keep steps at the **domain** level: `Given a film is in the catalogue`, not `Given I POST /films with JSON`.
- Reuse the same facade or `FilmApi` helpers as NUnit tests. Do not duplicate HTTP/serialization code in step definitions.
- Keep one binding class per feature area; use injected context objects for shared scenario state.
- Use Reqnroll for a few business-readable scenarios, not as a second copy of the module spec suite.

| | NUnit code specs | Reqnroll Gherkin |
| --- | --- | --- |
| Business can write/modify | No | Yes, in theory |
| Glue code cost | Low | Higher |
| Readability | High with Given/When/Then comments/names | High when maintained |
| Best for | Most module and acceptance tests | Scenarios the business truly owns |

Choose by **level of abstraction**, not ideology. Default to NUnit.

## 8. Readability and DSL

Ask of every line: **is this crucial to understanding the requirement?**

Before — noise:

```csharp
var imageX = new NewImageDto
{
    Categories = new[] { new Category("tree1", X, "AliasX") },
    OriginalFileName = "rambo-x.jpg"
};
_imageStore.Setup(x => x.Upload(It.IsAny<ImageToUpload>()))
    .ReturnsAsync(UploadedImage());
await Extract(_imageFacade.Save(imageX));
// ... repeated for Y and X+Y ...
```

After — requirement only:

```csharp
// given there are images in X, Y, and X+Y
var imageX = InCategory(X);
var imageY = InCategory(Y);
var imageXY = InCategory(X, Y);
ManagerHasImages(imageX, imageY, imageXY);

// when images are fetched by category X
var fetched = await Extract(_imageFacade.FindByCategoryId(X));

// then all images from X were retrieved
Assert.That(fetched.Select(x => x.OriginalFileName),
    Is.EqualTo(new[] { imageX, imageXY }.Select(x => x.OriginalFileName)));
```

If the test is about new releases, make that explicit:

```csharp
// wrong: the type is hidden and may change
var film = persistedFilms.First();

// right: the relevant fact is explicit
var film = persistedFilms.Single(x => x.FilmType == FilmType.NewRelease);
```

Build a small DSL when it clarifies the requirement. C# has no readable `C + G.at(1)`; use static factories and fluent extension methods instead.

```csharp
using static CategoryTree;

var root = A(B(), C(E(), F()), D());

var change = Add(G()).Under(C()).At(1);
var updated = ModifyTree(change);
var expected = Expected(A(B(), C(E(), G(), F()), D()));
Assert.That(updated, Is.EqualTo(expected));

// move B under F
var moved = ModifyTree(Move(B()).Under(F()));
Assert.That(moved, Is.EqualTo(Expected(A(C(E(), F(B())), D()))));
```

More examples:

```csharp
[Test]
public void Should_return_nearest_available_cars()
{
    // given cars A(2,2), B(1,1), C(2,2), D(400,400); A and B available
    var a = Car(2, 2);
    var b = Car(1, 1);
    var c = Car(2, 2);
    var d = Car(400, 400);
    LocationsDetected(a, b, c, d);
    Registered(a, b, c, d);
    Available(a, b);

    // when the user searches from (0,0)
    var found = FindCarsNearby(0, 0);

    // then nearest available cars are returned, sorted by distance
    Assert.That(found.Select(x => x.Id), Is.EqualTo(new[] { b.Id, a.Id }));
}
```

Balance abstraction: high enough to hide noise, low enough to debug. Build vocabulary, not a framework. If a helper hides the requirement, inline it.

## 9. Slow dependencies and test categories

| Dependency | .NET strategy |
| --- | --- |
| Core DB (Postgres/SQL Server) | Testcontainers + Respawn reset; keep in acceptance tests |
| Slow, rarely changes (Elastic) | External self-tested service/container; separate suite |
| Unstable vendor | WireMock.Net contract tests or a separate project/CI stage |
| Message broker | Testcontainers; in-memory transport for module specs |
| Time | `TimeProvider` / `IClock` with a fake |
| Other externals | Stub at the boundary (`HttpMessageHandler`) or contract test |

Move slow integration code outside the main build. Do not pay the price on every commit.

```csharp
[TestFixture]
[Category("Integration")]
public sealed class FilmAcceptanceTests { /* ... */ }
```

```sh
dotnet test --filter TestCategory!=Integration   # fast inner loop
dotnet test --filter TestCategory=Integration    # before merge/release
```

Use `[Explicit]` for tests that need manual setup or a live vendor.

## 10. Repairing an existing suite

1. **Baseline**: run `dotnet test`; record failures, time, flakiness.
2. **Classify**: module behavior, implementation test, end-to-end corner case, duplicated integration, slow vendor, redundant, genuine bug reproduction.
3. **Build a module harness**: in-memory configuration, samples, named actions.
4. **Migrate one behavior**: write the new NUnit module spec, delete the old test it replaces, run.
5. **Repeat** corner case by corner case.
6. **Extract interactions** for acceptance tests (`FilmApi`, typed clients).
7. **Remove I/O** from module specs; replace mocked internal adapters with in-memory ones.
8. **Replace internal-state assertions** with API results/events.
9. **Quarantine slow externals**; keep only critical paths in the main suite.
10. **Rename** tests to requirement sentences; add CI time budgets.

Common .NET repairs:

| Before | After |
| --- | --- |
| `new Mock<IFilmRepository>()` + `Verify(r => r.Save(...))` | `new FilmConfiguration().FilmFacade()` + assert `_facade.Show(title)` |
| Test-only `FilmFacade(IFilmRepository repository)` overload | One composition root; tests use the in-memory implementation |
| `[SetUp]` builds a 40-line DTO | `SampleFilm() with { Title = null! }` |
| Every test creates `HttpClient` and serializes JSON | `FilmApi` helper / typed client |
| `WebApplicationFactory` for domain behavior | Plain module configuration |
| One giant end-to-end test covering everything | Module specs for transitions + one publish acceptance path |
| `Assert.That(article.StatusField, ...)` | Returned DTO / `_facade.Show(id)` / published event |
| `DateTime.Now` in fixture and assertion | Fixed sample date + injected `TimeProvider` |
| `Thread.Sleep` / arbitrary delays | In-memory adapter or proper async assertion |
| Slow vendor call per build | WireMock.Net contract suite or external project |
| Test names like `Test1` | `Should_...` with Given/When/Then structure |
| Tests depend on execution order | Fresh fixture/reset DB per test |

Stop rules:

- Never delete a test without a replacement unless it is redundant or implementation-only.
- Migrate one module at a time; keep `dotnet test` green.
- Do not add a new test framework just to follow this skill.
- Do not chase coverage. Cover behaviors and corner cases.
- If the requirement is unclear, write the behavior matrix before code.

## 11. Review checklist

- [ ] Test is at the right level (module vs acceptance vs contract).
- [ ] Goes through the public API.
- [ ] Module spec has no `WebApplicationFactory`, `IServiceProvider`, EF Core, `HttpClient`, or sleeps.
- [ ] Internal adapters are in-memory; only other modules are mocked.
- [ ] Sample data is valid by default; overrides are minimal.
- [ ] Integration interactions have meaningful names (`FilmApi`, typed client).
- [ ] Assertions are observable, not internal.
- [ ] Corner cases are module-level; only critical paths use real I/O.
- [ ] Slow externals are outside the main build or tagged `[Category("Integration")]`.
- [ ] Reqnroll steps are domain-level and reuse helpers.
- [ ] Test fails for the right reason when behavior breaks.
- [ ] Suite is deterministic and independent of order.
- [ ] No duplicate coverage remains.

Aim for millisecond module specs, seconds for acceptance, minutes for the whole local suite.

## 12. Mantra

> Test modules. Test behavior, not implementation. Prepare sample data. Hide the API behind meaningful interactions. Build a small DSL. Explain it as at a whiteboard. Extract what is slow. Tests are specifications, and specifications are requirements.
