# ktor-arrow-example

RealWorld (Conduit) API — Medium-like blogging platform. Kotlin + Ktor + Arrow-kt.

## Stack

- **Ktor** — HTTP framework
- **Arrow-kt** — functional error handling (`Either`, `raise`, `ensure`)
- **Exposed** — SQL ORM (via persistence layer)
- **Kotest** — test framework with `SuspendFun` spec style
- **kotlinx-serialization** — JSON

## Architecture

Flat layered structure:

```
routes/       → HTTP layer (Ktor routes, request/response DTOs)
service/      → Business logic (interfaces + implementations)
repo/         → Persistence (interfaces + Exposed implementations)
auth/         → JWT authentication
DomainError.kt → All domain errors as sealed interface hierarchy
```

No circular imports: `routes` → `service` → `repo`. Never the other way.

## Error Handling

All errors flow through `DomainError` sealed interface. **Never throw in service or repo layers.**

```kotlin
// ✅ Correct — Arrow-kt Either
suspend fun register(input: RegisterUser): Either<DomainError, JwtToken> = either {
    ensure(input.username.isNotBlank()) { IncorrectInput(InvalidUsername(...)) }
    persistence.insert(input).bind()
}

// ❌ Wrong — throws
fun validate(input: String) {
    if (input.isBlank()) throw IllegalArgumentException("blank")
}
```

Use `raise` and `ensure` inside `either {}` blocks. Use `.bind()` to unwrap nested `Either`.

## Adding a New Feature

1. Add error types to `DomainError.kt` — new `sealed interface XError : DomainError`
2. Add service interface + data classes in `service/XService.kt`
3. Add persistence interface in `repo/XPersistence.kt`
4. Implement persistence with Exposed in `repo/XPersistenceImpl.kt` (or inline)
5. Add Ktor route in `routes/X.kt`
6. Wire up in DI (check how existing services are wired in `main.kt`)

## Tests

Tests use `SuspendFun` (custom Kotest spec) and real DB via `KotestProject`.

```kotlin
class UserServiceSpec : SuspendFun({
    val userService: UserService = KotestProject.dependencies.get().userService

    "register" - {
        "rejects blank username" {
            userService.register(RegisterUser("", "a@b.com", "pass")) shouldBeLeft IncorrectInput(...)
        }
        "returns token on success" {
            userService.register(RegisterUser("vini", "a@b.com", "pass")) shouldBeRight { token ->
                token.value.shouldNotBeBlank()
            }
        }
    }
})
```

**Write the test first. Run it, confirm it fails. Then implement.**

Use `shouldBeLeft` / `shouldBeRight` from `kotest-assertions-arrow`.

Run tests: `./gradlew test`
Run specific: `./gradlew test --tests "io.github.nomisrev.service.UserServiceSpec"`

## Arrow-kt Patterns Used Here

```kotlin
// either block — wraps a computation that can raise errors
val result: Either<DomainError, T> = either {
    val a = operationThatReturnsEither().bind()  // unwraps or short-circuits
    ensure(condition) { SomeDomainError }        // raises if false
    ensureNotNull(nullable) { SomeDomainError }  // raises if null
    a.doSomething()
}

// Combining multiple Eithers
val combined = zipOrAccumulate(validate1(), validate2()) { a, b -> Result(a, b) }
```

## Coding Rules

- `Either<DomainError, T>` return type on all service and persistence methods
- No `try/catch` inside service/repo — let it surface as `DomainError`
- New bounded contexts get their own error subtypes in `DomainError.kt`
- DTOs live in `routes/` — never leak into `service/` or `repo/`
- IDs are typed value classes (see `UserId` in `repo/`) — no raw `Long` leaking across layers
