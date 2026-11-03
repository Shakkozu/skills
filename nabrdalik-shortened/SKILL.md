---
name: Nabrdalik TDD (Short)
description: A compact guide to Jakub Nabrdalik's module-focused TDD for designing, writing, reviewing, and repairing tests. Use when choosing test levels, replacing brittle unit or end-to-end tests, or making a slow test suite fast and readable. Full version is the nabrdalik skill.
---

# Nabrdalik TDD — Short Version

> Source: Jakub Nabrdalik, "Improving your Test Driven Development".
> Companion to the full `nabrdalik` skill.

**Core idea:** test modules as black boxes through their public API, in milliseconds; add a few real-I/O acceptance tests for critical paths; make every test read like a requirement.

## 1. Test the right level

| Failure | Symptom | Fix |
| --- | --- | --- |
| **Too low** | Per-class tests with mocks; refactors break unrelated tests; 100% coverage still misses production bugs | Test module behavior through its facade |
| **Too high** | Whole system through UI/API; 3.5s empty context, 22s embedded Mongo, 45min suites | Push corner cases down to module specs |

A **module** is a vertical slice / bounded context: owns its data, exposes an API, hides its internals, and is a likely microservice candidate.

| Level | Scope | I/O | Speed | Count |
| --- | --- | --- | --- | --- |
| Module spec | One module through its facade | None / in-memory | Milliseconds | Many |
| Acceptance spec | Module through HTTP/DB/broker | Real core I/O | Seconds | Few |
| Contract spec | Slow/unstable third party | Real or stubbed external | Minutes | Outside main build |

Pure algorithms may be tested directly. Everything else is module behavior first.

## 2. Rules

1. Test **behavior**, not implementation.
2. Go through the **public API** — facade or HTTP endpoint.
3. **Do not let I/O out.** Wire in-memory adapters inside the module configuration; never expose a test-only repository parameter.
4. **Mock only other modules**, never the module's own repository/DAO/adapter.
5. Module specs use **no Spring, DB, network, filesystem, or sleeps**.
6. Use **sample data**: one valid default per DTO, override only what matters.
7. Extract **one obvious way** to interact with the API.
8. Write tests as **specifications**: Given/When/Then in domain language.
9. **Show, don't tell** — if you would draw it, express it in code.
10. **Remove before adding**; delete incidental detail.
11. Keep module specs in **milliseconds**, acceptance tests in **seconds**.
12. Tests == specifications == requirements.

## 3. Design before writing

For each behavior, fill a row:

| Given | When | Then | Level | I/O? | Sample data / DSL |
| --- | --- | --- | --- | --- | --- |
| a film is in the catalogue | it is requested by title | that film is returned | Module | No | `sampleFilm()` |
| no film has that title | it is requested | not-found error | Module | No | `sampleFilm(title: "x")` |
| two films exist | all films are requested over HTTP | both appear in JSON | Acceptance | Yes | `getFilms()` |

Then define:

- **samples** — `sampleFilm()`, `sampleArticle()`
- **named actions** — `postNewArticle(...)`, `preview(...)`
- **DSL** — `A(B, C(E, F), D)`, `C + G.at(1)`, `B >> F`
- **custom assertions** — `expectTree(updated).toEqual(...)`

If the vocabulary is hard to design, the requirement is not understood yet.

## 4. Module spec pattern

```java
class FilmConfiguration {
    FilmFacade filmFacade() {
        FilmRepository repo = new InMemoryFilmRepository();
        return new FilmFacade(repo, new FilmCreator());
    }
}
```

```groovy
class FilmSpec extends Specification {
    FilmFacade facade = new FilmConfiguration().filmFacade()

    def "should return a film by title"() {
        given: "a film is in the catalogue"
            FilmDto film = sampleFilm()
            facade.add(film)

        when: "the film is requested by title"
            FilmDto found = facade.show(film.title)

        then: "the system returns that film"
            found == film
    }
}
```

No Spring context. No repository passed in from the test. Add corner cases freely.

Java/JUnit:

```java
class FilmTest {
    private final FilmFacade facade = new FilmConfiguration().filmFacade();

    @Test
    @DisplayName("should return a film by title")
    void shouldReturnAFilmByTitle() {
        FilmDto film = sampleFilm();
        facade.add(film);
        assertThat(facade.show(film.title())).isEqualTo(film);
    }
}
```

## 5. Sample data, not setup explosions

```groovy
@CompileStatic
final class SampleFilms {
    private static final Map DEFAULTS = [
        title: "50 shades of Trumpet",
        type : FilmTypeDto.NEW
    ]

    static FilmDto sampleFilm(Map overrides = [:]) {
        Map p = DEFAULTS + overrides
        return FilmDto.builder()
            .title(p.title as String)
            .type(p.type as FilmTypeDto)
            .build()
    }
}

// sampleFilm(), sampleFilm(title: null), sampleFilm(type: OLD)
```

Rules:

- Default sample is **valid**; override only the field under test.
- Keep samples **deterministic**; inject a clock instead of using `now()`.
- Reuse the same sample data for integration JSON.
- If nested setup grows, add a named helper — not another 30-line builder chain.
- Sample factories live in test support, not production.

## 6. Integration specs

Keep real I/O at the edge. Extract every repeated interaction behind a meaningful name.

```groovy
trait OperatingOnFilmsEndpoint {
    abstract MockMvc getMockMvc()

    ResultActions getFilms() {
        mockMvc.perform(get("/films").accept(APPLICATION_JSON))
    }

    ResultActions getFilm(String title) {
        mockMvc.perform(get("/film/{title}", title).accept(APPLICATION_JSON))
    }
}
```

```groovy
class FilmAcceptanceSpec extends IntegrationSpec implements OperatingOnFilmsEndpoint {
    @Autowired FilmFacade facade

    def "should expose films over HTTP"() {
        given: "two films are in the catalogue"
            FilmDto a = sampleFilm(title: "A")
            FilmDto b = sampleFilm(title: "B")
            facade.add(a); facade.add(b)

        when: "the client asks for all films"
            ResultActions response = getFilms()

        then: "both are returned as JSON"
            response.andExpect(status().isOk())
                .andExpect(jsonPath('$.content[*].title',
                    containsInAnyOrder(a.title, b.title)))
    }
}
```

Developers should not re-decide endpoint, method, payload, serialization, or sync/async. Answer once; name methods after **domain actions** (`postNewArticle`, `preview`, `publish`), not HTTP verbs.

Assert observable results: DTOs, events, status/body, state read back through the API. Never assert repository calls, SQL, private fields, or logs.

Keep acceptance tests few: main happy path, integration contract, money path. Push corner cases down.

## 7. Readability and DSL

Ask of every line: **is this crucial to understanding the requirement?**

Before — noise:

```groovy
imageStore.upload(imageToUpload(imageX)) >> Mono.just(uploadedImage())
extract(imageFacade.save(imageX))
// ... repeated for every image ...
```

After — requirement only:

```groovy
given: "there are images in X, Y, and X+Y"
    imageX  = inCategory(X)
    imageY  = inCategory(Y)
    imageXY = inCategory(X, Y)
    managerHasImages(imageX, imageY, imageXY)

when: "images are fetched by category X"
    fetched = extract(imageFacade.findByCategoryId(X))

then: "all images from X were retrieved"
    fetched.originalFileName == [imageX, imageXY].originalFileName
```

If the test is about new releases, make that explicit:

```groovy
Film film = persistedFilms.find { it.filmType == NEW_RELEASE }
```

Build a small DSL when it clarifies the requirement:

```groovy
CategoryNode root = A(B, C(E, F), D)

given: "add G under C at position 1"
    AddNewCategoryDto change = C + G.at(1)
when: "the tree is modified"
    TreeDto updated = modifyTree(change)
then: "G appears between E and F"
    updated == expectedTree(A(B, C(E, G, F), D))

// move: modifyTree(B >> F)
```

In Groovy, `A(...)` is `call`, `+` is `plus`, `>>` is `rightShift`. In other languages, use builders or named functions: `add(g()).under(c()).at(1)`.

More examples:

```groovy
def "should return nearest available cars"() {
    given: "cars A(2,2), B(1,1), C(2,2), D(400,400); A and B available"
        A = car(2,2); B = car(1,1); C = car(2,2); D = car(400,400)
        locationsDetected(A, B, C, D)
        registered(A, B, C, D)
        available(A, B)

    when: "the user searches from (0,0)"
        found = findCarsNearby(0, 0)

    then: "nearest available cars are returned, sorted by distance"
        found*.id == [B, A]*.id
}
```

Balance abstraction: high enough to hide noise, low enough to debug. Build vocabulary, not a framework. If a helper hides the requirement, inline it.

## 8. Spec style and slow dependencies

- **Plain text** (Cucumber/JBehave): business can theoretically write scenarios, but glue code is slower and maintenance is harder. In practice business rarely owns them.
- **Code specs** (Spock/JUnit): faster to write and maintain; equally readable with disciplined Given/When/Then.
- Choose by **level of abstraction**, not ideology. Default to code specs.

When integration tests are slow:

| Dependency | Strategy |
| --- | --- |
| Core, fast, stable (Mongo) | Keep in integration tests |
| Slow, rarely changes (Elastic) | External self-tested jar + Docker; separate suite |
| Unstable vendor (AdWords) | Move out of build; half-automated + manual verification |
| Other externals | Stub at boundary or contract test |

Move slow integration code outside the build. Do not pay the price on every commit.

## 9. Repairing an existing suite

1. **Baseline**: run everything; record failures, time, flakiness.
2. **Classify**: module behavior, implementation test, end-to-end corner case, duplicated integration, slow vendor, redundant, genuine bug reproduction.
3. **Build a module harness**: in-memory configuration, samples, named actions.
4. **Migrate one behavior**: write the new module spec, delete the old test it replaces, run.
5. **Repeat** corner case by corner case.
6. **Extract interactions** for acceptance tests.
7. **Remove I/O** from module specs; replace mocked internal adapters with in-memory ones.
8. **Replace internal-state assertions** with API results/events.
9. **Quarantine slow externals**; keep only critical paths in the main suite.
10. **Rename** tests to requirement sentences; add CI time budgets.

Common repairs:

| Before | After |
| --- | --- |
| `new FilmConfiguration().filmFacade(repositoryMock)` + `verify(save)` | `new FilmConfiguration().filmFacade()` + assert `facade.show(title)` |
| Every test builds MockMvc request/serialization | `postNewArticle(...)`, `preview(...)` helpers |
| 40-line DTO builder, one field changed | `sampleNewArticle(title: null)` |
| One giant E2E test covering everything | Module specs for transitions + one publish acceptance path |
| `assertThat(article.statusField)...` | `facade.show(id)` / returned DTO / published event |
| `now()` in fixture and assertion | Fixed sample date + injected `Clock` |
| Slow vendor call per build | External contract suite |

Stop rules:

- Never delete a test without a replacement unless it is redundant or implementation-only.
- Migrate one module at a time; keep the suite green.
- Do not add a new framework just to follow this skill.
- Do not chase coverage. Cover behaviors and corner cases.
- If the requirement is unclear, write the behavior matrix before code.

## 10. Review checklist

- [ ] Test is at the right level (module vs acceptance vs contract).
- [ ] Goes through the public API.
- [ ] Module spec has no Spring/DB/network/sleeps.
- [ ] Internal adapters are in-memory; only other modules are mocked.
- [ ] Sample data is valid by default; overrides are minimal.
- [ ] Integration interactions have meaningful names.
- [ ] Assertions are observable, not internal.
- [ ] Corner cases are module-level; only critical paths use real I/O.
- [ ] Slow externals are outside the main build.
- [ ] Test fails for the right reason when behavior breaks.
- [ ] Suite is deterministic and independent of order.
- [ ] No duplicate coverage remains.

Aim for millisecond module specs, seconds for acceptance, minutes for the whole local suite.

## 11. Mantra

> Test modules. Test behavior, not implementation. Prepare sample data. Hide the API behind meaningful interactions. Build a small DSL. Explain it as at a whiteboard. Extract what is slow. Tests are specifications, and specifications are requirements.
