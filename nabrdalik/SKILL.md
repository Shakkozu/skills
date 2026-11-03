---
name: Nabrdalik TDD
description: Design, write, review, and repair automated tests using Jakub Nabrdalik's module-focused TDD. Use when choosing test levels, replacing brittle unit or end-to-end tests, writing module or acceptance specs, building sample data and test DSLs, or fixing a slow/flaky test suite.
---

# Nabrdalik TDD: Module-Focused Test Design

> Source: Jakub Nabrdalik, "Improving your Test Driven Development" (96 slides).
> This skill turns the talk into an operating manual.

The central claim: **test modules as black boxes through their public API, in milliseconds; add a few real-I/O acceptance tests for the paths that matter; make every test read like a requirement.**

---

## 1. The core model

### Two failure modes

| Failure | What it looks like | Why it hurts |
| --- | --- | --- |
| **Test too low** | Unit-testing every method and class, mocking collaborators | Any API change, refactor, or feature breaks unrelated tests. 100% coverage still misses production failures because it tests implementation, not system behavior. |
| **Test too high** | Whole system through UI/API for every case | Empty Spring context costs ~3.5 s; a microservice with embedded Mongo ~22 s; a full suite can reach 45 minutes. Slow feedback kills TDD. |

### The answer: test modules

A **module**:

- encapsulates its data; access only through its API
- has clearly defined collaborators and APIs
- contains (almost) all layers — a vertical slice
- is a likely microservice candidate
- is usually a **Bounded Context** (its words have their own meaning)

Test each module **as a black box**:

- all flows and corner cases in **milliseconds, with no I/O**
- plus a small number of **critical, money-making paths with real I/O**

### Test levels

| Level | Scope | I/O | Speed | How many |
| --- | --- | --- | --- | --- |
| **Module spec** | One module through its facade/API | None, or in-memory adapters | Milliseconds | Many — cover every flow and corner case |
| **Acceptance / integration spec** | Module through HTTP, DB, broker | Real core I/O | Seconds | Few — critical paths and contracts |
| **External contract spec** | Slow or unstable third parties | Real or stubbed external system | Minutes | Outside the main build |

Do not default to per-class unit tests. A pure algorithm with no collaborators may be tested directly, but the primary unit of design is the **module behavior**, not the class.

---

## 2. Non-negotiable rules

1. **Test behavior, not implementation.** Assert what the module does for its users, not which internal methods ran.
2. **Go through the public API.** Module specs call the facade; acceptance specs call the HTTP endpoint. Do not reach into repositories or private state.
3. **Do not let I/O out of the module.** A module's in-memory adapters are wired inside its configuration. Never expose a test-only configuration method that accepts a repository, because developers will mock it and verify internal state.
4. **Mock only other modules.** If a module collaborates with another module, inject that module's facade (or better, consume its events). Never mock the module's own repository, DAO, or adapter in a module spec.
5. **Keep module specs free of Spring, DB, network, filesystem, sleeps, and clock waits.** An empty Spring context is already too slow for the inner loop.
6. **Use sample data, not setup explosions.** Every module gets a valid default sample DTO; each test overrides only the field that matters.
7. **Extract one obvious way to interact with the API.** Developers should not re-decide endpoint, method, payload, serialization, or sync/async in every test.
8. **Write tests as specifications.** Use Given/When/Then names in domain language; a reader should understand the requirement without reading production code.
9. **Show, don't tell.** If you would draw it on a whiteboard, express it in code — build a small test DSL when it clarifies the requirement.
10. **Remove before adding.** Perfection is reached when there is nothing left to remove. Delete incidental detail from tests.
11. **Keep the fast suite fast.** Module specs in milliseconds; acceptance specs in seconds; the whole local suite in minutes, not tens of minutes.
12. **Tests == specifications == requirements.** Refining requirements is hard; make the scenarios explicit before writing code.

---

## 3. Workflow: design tests before writing them

### Step 1 — Identify the module and its API

Answer:

- What is the module's name and bounded context?
- What is its public facade or HTTP API?
- What data does it own?
- Which collaborators are other modules, and which are internal adapters?
- What observable outcomes prove the behavior: returned DTO, event, HTTP response, persisted-then-read state?

### Step 2 — Build a behavior matrix

Write the scenarios before writing test code. Keep one row per behavior.

| # | Given | When | Then | Level | I/O? | Sample data / DSL |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | a film is in the catalogue | the film is requested by title | that film is returned | Module | No | `sampleFilm()` |
| 2 | no film has that title | the film is requested | not-found error | Module | No | `sampleFilm(title: "missing")` |
| 3 | two films are in the catalogue | all films are requested over HTTP | both appear in JSON | Acceptance | Yes | `sampleFilm()`, `getFilms()` |
| 4 | a category tree exists | a category is moved | the tree matches the expected shape | Module | No | `A(B, C(E, F), D)`, `B >> F` |

Rules for the matrix:

- Cover **flows and corner cases** at module level.
- Mark only **critical paths** (the ones that bring money or prove the integration) as acceptance tests.
- If a scenario needs a slow external system to be meaningful, decide whether it belongs in a separate contract suite.
- Turn ambiguous requirements into questions now, not into test code later.

### Step 3 — Design the vocabulary

Before writing tests, define:

- **Sample data factories**: `sampleFilm()`, `sampleArticle()`, `sampleNewArticle(title: null)`.
- **Action helpers**: `postNewArticle(...)`, `previewArticle(...)`, `moveCategory(...)`.
- **Domain DSL**: `A(B, C(E, F), D)`, `C + G.at(1)`, `B >> F`, `car(2, 2)`.
- **Custom assertions**: `expectTree(updated).toEqual(A(B, C(E, G, F), D))`.

If the vocabulary is hard to design, the requirement is probably not understood yet.

### Step 4 — Write the first failing module spec

Start at the facade. Make it fail for the right reason. Then implement the minimum and refactor.

### Step 5 — Add I/O at the edge

Only after the module behavior works in memory:

- add the real repository/adapter
- add the controller or message listener
- add a thin acceptance spec for the critical path
- keep the module spec on the in-memory configuration

---

## 4. Workflow: write a module spec

### 4.1 Module configuration

The module configuration is the composition root. It wires the facade and its internal adapters. Tests construct it directly.

```java
class FilmConfiguration {
    FilmFacade filmFacade() {
        FilmRepository repository = new InMemoryFilmRepository();
        return new FilmFacade(repository, new FilmCreator());
    }
}
```

```groovy
class FilmSpec extends Specification {
    FilmFacade facade = new FilmConfiguration().filmFacade()
}
```

No Spring context. No repository passed in from the test.

### 4.2 First behavior

```groovy
def "should return a film by title"() {
    given: "a film is in the catalogue"
        FilmDto film = sampleFilm()
        facade.add(film)

    when: "the film is requested by title"
        FilmDto found = facade.show(film.title)

    then: "the system returns that film"
        found == film
}
```

### 4.3 Corner cases

Add them freely. They are cheap.

```groovy
def "should reject a film that does not exist"() {
    when: "a missing film is requested"
        facade.show("missing title")

    then: "the system reports that it is not found"
        thrown(FilmNotFoundException)
}
```

### 4.4 Refactor

Green tests let you refactor the module internals. Because tests go through the facade, renaming a class, splitting a service, or changing the in-memory repository does not break the specification.

### 4.5 Java/JUnit equivalent

```java
class FilmTest {
    private final FilmFacade facade = new FilmConfiguration().filmFacade();

    @Test
    @DisplayName("should return a film by title")
    void shouldReturnAFilmByTitle() {
        // given
        FilmDto film = sampleFilm();
        facade.add(film);

        // when
        FilmDto found = facade.show(film.title());

        // then
        assertThat(found).isEqualTo(film);
    }
}
```

---

## 5. Test data: samples, not setups

### The pattern

For each module, prepare:

- one **valid default** input DTO
- one **valid default** output DTO if useful
- a factory that allows overriding only the fields a test cares about
- the same sample data serialized to JSON for integration tests

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
```

Usage:

```groovy
FilmDto normal   = sampleFilm()
FilmDto noTitle  = sampleFilm(title: null)
FilmDto oldFilm  = sampleFilm(type: FilmTypeDto.OLD)
```

Java version:

```java
final class SampleFilms {
    static FilmDto sampleFilm(UnaryOperator<FilmDtoBuilder> customizer) {
        return customizer.apply(FilmDto.builder()
                .title("50 shades of Trumpet")
                .type(FilmTypeDto.NEW))
            .build();
    }

    static FilmDto sampleFilm() {
        return sampleFilm(b -> b);
    }
}

// sampleFilm(b -> b.title(null));
```

### Rules

- The default sample is **valid**; a test overrides only the field under test.
- No test should build the same 20-field object from scratch.
- Keep samples deterministic: stable titles, IDs, dates. Inject a clock when time matters; never depend on `now()` or random values.
- Share the sample map with integration tests so JSON payloads stay in sync.
- Keep sample factories in test support code, not production code.
- If a sample needs a lot of nested data, add a named helper (`sampleArticleWithCategory(...)`) rather than repeating the structure.

---

## 6. Integration / acceptance specs

### 6.1 Keep I/O at the edge

The module spec uses the in-memory configuration. The acceptance spec uses the real configuration: real HTTP layer, real database if the database is central to the behavior, real broker if messaging is the point.

### 6.2 Extract common interactions

Do not make a developer think again about:

- what endpoint to hit
- what protocol or HTTP method to use
- what payload to send
- how to serialize it
- how to deserialize the response
- whether the call is sync or async

Answer those questions once, behind meaningful names.

```groovy
trait OperatingOnFilmsEndpoint {
    abstract MockMvc getMockMvc()

    ResultActions getFilms() {
        return mockMvc.perform(get("/films").accept(APPLICATION_JSON))
    }

    ResultActions getFilm(String title) {
        return mockMvc.perform(get("/film/{title}", title).accept(APPLICATION_JSON))
    }
}
```

```groovy
trait CommunicatingWithEndpoint {
    abstract MockMvc getMockMvc()

    ResultActions performGet(String path) {
        return mockMvc.perform(get(path).accept(APPLICATION_JSON))
    }

    ResultActions performAsyncGet(String path) {
        MvcResult result = mockMvc.perform(get(path).accept(APPLICATION_JSON)).andReturn()
        return mockMvc.perform(asyncDispatch(result))
    }

    Object getObjectFromResponse(ResultActions response) {
        return new JsonSlurper().parseText(
            response.andReturn().response.contentAsString)
    }
}
```

Name methods after **domain actions**, not HTTP verbs: `postNewArticle`, `updateExistingArticle`, `preview`, `submit`, `publish`, `getFilms`.

### 6.3 Acceptance skeleton

```groovy
class FilmAcceptanceSpec extends IntegrationSpec implements OperatingOnFilmsEndpoint {
    @Autowired FilmFacade facade

    def "should expose films over HTTP"() {
        given: "two films are in the catalogue"
            FilmDto trumper = sampleFilm(title: "50 shades of Trumpet")
            FilmDto clingon = sampleFilm(title: "American Clingon Bondage")
            facade.add(trumper)
            facade.add(clingon)

        when: "the client asks for all films"
            ResultActions response = getFilms()

        then: "both films are returned as JSON"
            response.andExpect(status().isOk())
                .andExpect(jsonPath('$.content[*].title',
                    containsInAnyOrder(trumper.title, clingon.title)))
    }
}
```

### 6.4 What to assert

- observable API results: DTOs, events, HTTP status and body
- persisted state **read back through the API**
- contract details that clients depend on
- unchanged state when the requirement says "and everything else stays the same"

Do **not** assert:

- repository calls (`verify(repository).save(...)`)
- SQL, table rows, or internal collections
- private fields, logs, or implementation order
- every field of a DTO when only two fields matter to the requirement

### 6.5 Keep acceptance tests few

One or a few per module, focused on:

- the main happy path
- the integration contract (JSON shape, status codes)
- the path that would cost money if broken

Push every corner case down to a module spec.

---

## 7. Readability: show, don't tell

### 7.1 Remove incidental detail

Ask of every line: **is this crucial to understanding the requirement?**

Before — noise from stubbing, repeated DTO construction, and plumbing:

```groovy
def "should find images by category"() {
    given: "image in category X"
        NewImageDto imageX = newImageDto([
            categories: [new Category("tree1", X, "AliasX")],
            originalFileName: "rambo-x.jpg"
        ])
        imageStore.upload(imageToUpload(imageX)) >> Mono.just(uploadedImage())
        extract(imageFacade.save(imageX))
    and: "there is an image in category Y"
        // ... fifteen more lines ...
    when: "images are fetched by category X"
        List<ImageDto> fetched = extract(imageFacade.findByCategoryId(X))
    then:
        fetched.originalFileName == [imageX, imageXY].originalFileName
}
```

After — only the requirement:

```groovy
def "should find images by category"() {
    given: "there are three images: in category X, in Y, in X and Y"
        NewImageDto imageX  = inCategory(X)
        NewImageDto imageY  = inCategory(Y)
        NewImageDto imageXY = inCategory(X, Y)
        managerHasImages(imageX, imageY, imageXY)

    when: "images are fetched by category X"
        List<ImageDto> fetched = extract(imageFacade.findByCategoryId(X))

    then: "all images from category X were retrieved"
        fetched.originalFileName == [imageX, imageXY].originalFileName
}
```

### 7.2 Explicit vs implicit information

- **Explicit**: what matters for this particular test.
- **Implicit**: what can be taken for granted.

If the test is about new releases, do not grab an arbitrary film:

```groovy
// wrong: the type is hidden and may change
Film film = persistedFilms.first()

// right: the relevant fact is explicit
Film film = persistedFilms.find { it.filmType == NEW_RELEASE }
```

If preparing the assumption takes logic, move it into a named helper:

```groovy
given:
    Article unpublishedArticle = createUnpublishedArticle()
```

### 7.3 Build a small DSL

If you would draw the scenario on a whiteboard, try to write that drawing in code.

**Tree example.** Declare a tree, add a node, compare the expected tree:

```groovy
CategoryNode root = A(
    B,
    C(E, F),
    D
)

def "should add a new category"() {
    given: "the user adds G under C at position 1"
        AddNewCategoryDto change = C + G.at(1)

    when: "the tree is modified"
        TreeDto updated = modifyTree(change)

    then: "G appears between E and F"
        updated == expectedTree(
            A(
                B,
                C(E, G, F),
                D
            )
        )
}
```

Move a category with a readable operator:

```groovy
def "should move category under a different category"() {
    when:
        TreeDto updated = modifyTree(B >> F)

    then:
        updated == expectedTree(
            A(
                C(
                    E,
                    F(B)
                ),
                D
            )
        )
}
```

In Groovy, `A(...)` is `call`, `C + G` is `plus`, `G.at(1)` is a method, and `B >> F` is `rightShift`. The DSL is just methods with domain names.

**In a language without operator overloading**, keep the same vocabulary with builders or named functions:

```java
CategoryTree before = tree(a(b(), c(e(), f()), d()));
AddCategory change = add(g()).under(c()).at(1);
CategoryTree updated = modifyTree(before, change);
assertThat(updated).isEqualTo(tree(a(b(), c(e(), g(), f()), d())));
```

**Car search example.** Coordinates and named actions beat raw setup:

```groovy
def "should return nearest available cars"() {
    given: "cars A(2,2), B(1,1), C(2,2), D(400,400); only A and B are available"
        Car A = car(2, 2)
        Car B = car(1, 1)
        Car C = car(2, 2)
        Car D = car(400, 400)
        locationsDetected(A, B, C, D)
        registered(A, B, C, D)
        available(A, B)

    when: "the user searches from (0,0)"
        List<Car> found = findCarsNearby(0, 0)

    then: "the nearest available cars are returned, sorted by distance"
        found*.id == [B, A]*.id
}
```

### 7.4 Balance the abstraction

- High enough to hide incidental detail and focus on the requirement.
- Low enough that a failure is easy to diagnose and the abstraction does not leak.
- Build vocabulary, not a framework.
- If a helper hides the requirement, inline it.
- If setup repeats three times or obscures intent, extract it.

---

## 8. Plain-text specs vs code specs

| | Plain text (Cucumber/JBehave) | Code (Spock/JUnit) |
| --- | --- | --- |
| Business can write/modify | Yes, in theory | No |
| Glue code cost | Higher; slower and harder | Lower |
| Readability | High when maintained | Equally high with disciplined Given/When/Then |
| Maintenance | Slower | Faster |
| Best when | Business genuinely owns and edits scenarios | Most engineering teams |

**Recommendation:** default to code specs with Given/When/Then strings. Use plain-text specs only when the business really collaborates on scenarios. In practice, business rarely maintains them.

The real question is not "Cucumber or Spock?" but **what level of abstraction does the test need?** Keep it high enough to focus on important information, low enough to make the work easy. DSLs and factory methods are how you tune that level.

---

## 9. Slow suites and external systems

When integration tests are still slow, classify each dependency:

| Dependency | Strategy |
| --- | --- |
| Core, fast, stable (e.g. Mongo) | Keep it inside integration tests; it participates in most operations |
| Slow, rarely changes (e.g. Elastic) | Move to an external self-tested jar + Docker; run its tests separately |
| Unstable vendor with no sensible test strategy (e.g. AdWords) | Move out of the build; use half-automated tests and manual UI verification |
| Everything else | Stub at the module boundary or use a contract test |

Rule: **move code with slow integration tests outside your build, if possible.** Do not pay the price on every commit.

Also:

- Tag slow tests and run them on a schedule or before release.
- Measure per-test time; investigate anything far above the budget.
- Keep the main suite fast enough that developers run it voluntarily.
- Do not "fix" slowness by deleting the only critical-path test. Replace it with a cheaper test at the right level.

---

## 10. Repairing an existing test suite

### 10.1 Baseline first

1. Run the whole suite. Record failures, total time, and flaky tests.
2. Do not refactor on red unless the test itself is wrong.
3. Save one green baseline commit if possible.

### 10.2 Classify every test

| Class | Action |
| --- | --- |
| Module behavior + public API | Keep; improve names, data, assertions |
| Method/class-level with mocks of own module | Replace with module specs; delete redundant |
| End-to-end for a corner case | Push down to a module spec; keep one acceptance path |
| Integration with duplicated HTTP setup | Extract interaction helpers; migrate |
| Slow vendor integration | Quarantine; contract test or external suite |
| Implementation assertion (SQL, repository calls, private state) | Rewrite around observable behavior |
| Redundant duplicate coverage | Delete after the replacement exists |
| Genuine bug reproduction | Keep, but move to the right level and name the requirement |

### 10.3 Repair order

Work in thin vertical slices, one module at a time.

1. **Create the module harness**: in-memory configuration, sample data, named actions.
2. **Write one new module spec** for a behavior currently covered by brittle tests.
3. **Delete the old tests it replaces**; run the new test.
4. **Repeat behavior by behavior**, adding corner cases cheaply.
5. **Extract the interaction layer** for acceptance tests.
6. **Remove I/O from module specs**; replace mocks of internal adapters with in-memory ones.
7. **Replace internal-state assertions** with API results, events, or read-back-through-the-API.
8. **Quarantine slow externals**; keep only critical paths in the main suite.
9. **Rename tests** to requirement sentences with Given/When/Then structure.
10. **Add CI budgets** for the fast suite and for acceptance tests.

### 10.4 Common repairs

**Mocked repository → in-memory module configuration**

Before:

```groovy
def "test add film"() {
    FilmRepository repository = Mock()
    FilmFacade facade = new FilmConfiguration().filmFacade(repository) // I/O leaked

    facade.add(sampleFilm())

    verify(repository).save(_) // asserts implementation
}
```

After:

```groovy
def "should add a film to the catalogue"() {
    given: "a valid film"
        FilmDto film = sampleFilm()

    when: "the film is added"
        facade.add(film)

    then: "it can be retrieved by title"
        facade.show(film.title) == film
}
```

**Duplicated HTTP setup → named interaction**

Before: every test builds a `MockMvc` request, sets content type, serializes JSON, and parses the response.

After: `postNewArticle(...)`, `preview(article.id, articleJson())`, `publish(...)` — one definition, meaningful names.

**Setup explosion → sample data + helper**

Before: 40 lines of builders for a valid article, then one field changed.

After:

```groovy
NewArticleDto article = sampleNewArticle(title: null)
```

**Giant end-to-end → module specs + one acceptance path**

Before: one test walks bio creation, article creation, update, preview, submit, publish, and asserts everything.

After:

- module specs for each state transition and corner case
- one acceptance test for the publish flow and its HTTP contract
- shared sample data and interaction helpers

**Internal state assertion → observable outcome**

Before: `verify(repository).delete(id)` or `assertThat(article.statusField).isEqualTo(...)`.

After: `facade.show(id)` throws not-found, or the returned DTO has `status == DELETED`, or the expected event was published.

**Brittle time/randomness → injectable clock and deterministic samples**

Before: `publishDate = now()` in the fixture and an assertion against `now()`.

After: fixed `publishDate` in the sample, `Clock` fixed in the module configuration.

### 10.5 Stop rules

- Never delete a test without a replacement, unless it is genuinely redundant or asserts implementation details.
- Do not rewrite the whole suite in one branch; migrate one module at a time.
- Do not introduce a new framework or dependency just to satisfy this skill; use the project's existing idioms.
- Do not chase a coverage number. Cover behaviors and corner cases with meaningful assertions.
- If the requirement is unclear, stop and write the behavior matrix first. Refining requirements is a hard job; do not hide it inside test code.

---

## 11. Review rubric

Score each criterion 0–2. Target: **16+/20**.

| Criterion | 0 | 1 | 2 |
| --- | --- | --- | --- |
| **Test level** | Per-class or whole-system by default | Mixed, inconsistent | Module specs for behavior; few acceptance tests for critical paths |
| **API boundary** | Tests internals or mocks own adapters | Partly through the facade | Always through the module's public API |
| **Behavior focus** | Asserts calls, SQL, private state | Mixed | Asserts observable outcomes |
| **Readability** | Cryptic names, plumbing noise | Some domain names | Given/When/Then, one behavior, domain vocabulary |
| **Test data** | Ad-hoc builders everywhere | Some factories | Valid samples with minimal overrides |
| **I/O discipline** | Spring/DB/network in every test | Slow areas remain | No I/O in module specs; real I/O only where it matters |
| **Integration ergonomics** | Duplicated HTTP/serialization code | Partial helpers | Named interactions; serialization answered once |
| **Speed** | Tens of minutes, flaky | Slow pockets | Millisecond module specs; seconds for acceptance |
| **Duplication** | Several ways to test the same thing | Some overlap | One obvious way per interaction |
| **Spec value** | Test names only | Partial requirements | Tests read as requirements; corner cases explicit |

Use the rubric in code review and when proposing a repair plan. Report the score before and after.

---

## 12. Smells → fixes

| Smell | Underlying problem | Fix |
| --- | --- | --- |
| Test breaks when a class is renamed | Testing implementation | Move to a facade-level module spec |
| `verify(repository).save(...)` | Internal state as the oracle | Assert the API result or event |
| `@SpringBootTest` for domain logic | Test too high | Plain module configuration; no context |
| 100% line coverage, bugs in production | Coverage of implementation, not behavior | Behavior matrix; corner cases through the API |
| Every test rebuilds a 30-field DTO | Setup explosion | Sample factory with overrides |
| `first()` / arbitrary fixture selection | Implicit information | Query for the property the test needs |
| `Thread.sleep` / `await` with timeout | Real async I/O in the fast suite | In-memory adapter or proper async assertion |
| Duplicated MockMvc boilerplate | No interaction layer | Trait/client with named actions |
| One test asserts ten unrelated things | No single behavior | Split into focused specs; keep a narrative only when it tells one story |
| Test names like `testAdd1` | Tests are not specifications | Rename to `should ... when ...` in domain language |
| Slow vendor call on every build | Paying the wrong price | Extract to self-tested jar/Docker or external contract suite |
| Random fixtures | Non-deterministic failures | Fixed samples; injected clock/seed |
| Private field assertions | Leaking abstraction | Assert through the public API or an event |
| Tests depend on execution order | Shared mutable state | Isolate module configuration and sample data per test |

---

## 13. How to respond when acting as this expert

### When asked to design tests

Return, in order:

1. **Module and owner** — what module owns the behavior, what its API is.
2. **Behavior matrix** — Given/When/Then rows, including corner cases, with level and I/O needs.
3. **Test-level plan** — which scenarios are module specs, which are acceptance/contract tests, and why.
4. **Test data and DSL plan** — sample factories, named actions, custom assertions.
5. **Risks and open questions** — ambiguous requirements, missing APIs, slow dependencies.

Do not start writing code until the matrix is coherent.

### When asked to write tests

- Read the existing test conventions, build tool, and module structure first.
- Implement the smallest behavior at the right level.
- Use the project's existing framework and idioms.
- Run the tests; report the exact command and result.
- Refactor production code only as much as the failing test demands.

### When asked to fix existing tests

- Baseline the suite and classify tests using §10.
- Propose a prioritized, incremental repair plan.
- Migrate one module/behavior at a time; keep the suite green.
- Show before/after for at least one representative test.
- Report the rubric score and the new suite time when done.

### When asked to review tests

- Cite the rubric criterion and the exact test lines.
- Distinguish "wrong level" from "wrong assertion" from "unreadable setup".
- Prefer concrete rewrites over abstract advice.
- Do not demand a framework change when a helper or a sample factory will do.

---

## 14. Definition of done

- [ ] The behavior is named in domain language, not implementation language.
- [ ] The test goes through the module's public API.
- [ ] Module specs have no Spring context, DB, network, filesystem, or sleeps.
- [ ] Internal adapters are in-memory and wired by the module configuration.
- [ ] Only other modules are mocked, never the module's own adapters.
- [ ] Sample data is valid by default; each test overrides only what matters.
- [ ] Integration interactions have meaningful names; serialization is not repeated.
- [ ] Assertions are on observable outcomes, not internal state.
- [ ] Corner cases live at module level; only critical paths use real I/O.
- [ ] Slow/unstable externals are outside the main build or clearly tagged.
- [ ] The test fails for the right reason when the behavior is broken.
- [ ] The suite is deterministic and independent of execution order.
- [ ] The fast suite stays in milliseconds per module spec and seconds per acceptance test.
- [ ] No redundant tests remain after migration.

---

## 15. Source map

| Slides | Topic |
| --- | --- |
| 7–11 | Test too low / too high; test modules as black boxes |
| 12–19 | Module specs through the facade; in-memory adapters; fast feedback |
| 20–28 | I/O at the edge; do not let I/O out; module boundaries; behavior over internal state |
| 29–41 | Minimize information; explicit vs implicit; sample data for modules |
| 42–51 | Common interactions; extract HTTP/serialization once; meaningful names |
| 52–74 | Show, don't tell; whiteboard approach; test DSLs; remove before adding |
| 75–84 | Plain-text vs code specs; level of abstraction |
| 85–87 | Slow integration tests; external jars/Docker; move slow code out of the build |
| 88–94 | Natural-language specs; requirements refinement; summary |

---

## 16. The mantra

> Focus on testing modules. Test behavior, not implementation. Prepare sample data. Hide the API behind meaningful interactions. Build a small DSL. Explain it as you would at a whiteboard. Extract what is slow. Tests are specifications, and specifications are requirements.
