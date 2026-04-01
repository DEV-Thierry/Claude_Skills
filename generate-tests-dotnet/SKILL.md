---
name: generate-tests-dotnet
description: >
  Generates comprehensive tests for .NET Clean Architecture + CQRS (MediatR) projects.
  Use this skill whenever the user says "gera testes para o módulo X", "generate tests for X",
  "cria testes", "quero testes para", or any variation asking to test a module/feature in a
  .NET project. The skill auto-discovers module files, generates Functional Tests and Unit Tests
  following existing project conventions, then compiles and auto-fixes any errors.
  Default project is PulseBoard, but works for any .NET Clean Architecture project.
---

# Generate Tests for a .NET Clean Architecture Module

## Overview

This skill generates comprehensive tests for a given module in a .NET project that follows
Clean Architecture with CQRS (MediatR). It covers:

1. **Discovery** — find all relevant source files for the module
2. **Functional Tests** — integration tests dispatched via MediatR
3. **Unit Tests** — validator unit tests when complex logic is present
4. **Build & Fix** — compile and auto-correct any errors

---

## Step 1: Identify Project Context

Before generating anything, establish the project configuration:

1. **Project name & namespace**: Default is `PulseBoard`. If the user mentions a different project
   or the codebase uses a different root namespace, adapt all namespaces accordingly.

2. **Locate the test infrastructure** by reading:
   - `tests/Application.FunctionalTests/Infrastructure/TestBase.cs` — base class all tests inherit from
   - `tests/Application.FunctionalTests/GlobalUsings.cs` — namespaces already globally imported (do NOT re-add these in test files)
   - `tests/Application.FunctionalTests/Infrastructure/TestApp.cs` (or similar) — available helper methods

3. **Confirm the module name** from `$ARGUMENTS`. If not provided, ask the user.

> **Adapting to other projects**: If namespace, folder structure, or test helper names differ
> from PulseBoard defaults, adjust every generated file accordingly. The patterns below use
> PulseBoard conventions — treat them as templates, not hard-coded values.

---

## Step 2: Discover the Module

Run these discoveries in parallel:

```
src/Application/{ModuleName}/Commands/        → all Command + Handler + Validator files
src/Application/{ModuleName}/Queries/         → all Query + Handler + DTO files
src/Application/{ModuleName}/EventHandlers/   → any domain event handlers (optional)
src/Domain/Entities/{EntityName}.cs           → the domain entity
src/Application/Common/Interfaces/IApplicationDbContext.cs → check for DbSet<{Entity}>
```

Read **every file found** — commands, queries, handlers, validators, DTOs. You need to know:
- All fields and their validation rules
- Whether commands are decorated with `[Authorize]` or `[Authorize(Roles = "...")]`
- Whether validators contain async rules or complex custom logic
- Whether the entity's DbSet exists in `IApplicationDbContext` (warn the user if missing)

---

## Step 3: Generate Functional Tests

### File locations

```
tests/Application.FunctionalTests/{ModuleName}/Commands/{CommandName}Tests.cs
tests/Application.FunctionalTests/{ModuleName}/Queries/{QueryName}Tests.cs
```

### Rules (apply to every generated test file)

- Every test class **must** inherit from `TestBase`
- Namespace must match folder: `{RootNamespace}.Application.FunctionalTests.{ModuleName}.Commands`
- Use `TestApp.SendAsync()` to dispatch commands/queries
- Use `TestApp.FindAsync<TEntity>()` to verify DB state
- Use `TestApp.AddAsync<TEntity>()` to seed data
- Use `TestApp.RunAsDefaultUserAsync()` / `TestApp.RunAsAdministratorAsync()` for auth context
- Use `TestApp.CountAsync<TEntity>()` for count checks
- Do **not** add `using` statements for namespaces in `GlobalUsings.cs`

### Test cases per command type

**Create commands:**
```csharp
// 1. Minimum fields validation
[Test]
public async Task ShouldRequireMinimumFields()
{
    var command = new CreateXCommand();
    await Should.ThrowAsync<ValidationException>(() => TestApp.SendAsync(command));
}

// 2. Happy path — assert all fields + audit properties
[Test]
public async Task ShouldCreateX()
{
    var userId = await TestApp.RunAsDefaultUserAsync();
    var command = new CreateXCommand { /* valid data */ };
    var id = await TestApp.SendAsync(command);
    var item = await TestApp.FindAsync<X>(id);

    item.ShouldNotBeNull();
    item!.Title.ShouldBe(command.Title);
    // ... other fields
    item.CreatedBy.ShouldBe(userId);
    item.Created.ShouldBe(DateTime.Now, TimeSpan.FromMilliseconds(10000));
    item.LastModifiedBy.ShouldBe(userId);
    item.LastModified.ShouldBe(DateTime.Now, TimeSpan.FromMilliseconds(10000));
}

// 3. One test per validation rule (MaxLength, Required, Unique, etc.)
```

**Update commands:**
```csharp
[Test]
public async Task ShouldRequireValidXId()
{
    var command = new UpdateXCommand { Id = 99 };
    await Should.ThrowAsync<NotFoundException>(() => TestApp.SendAsync(command));
}

[Test]
public async Task ShouldUpdateX()
{
    var userId = await TestApp.RunAsDefaultUserAsync();
    // seed → update → assert updated fields + LastModifiedBy/LastModified
}
```

**Delete commands:**
```csharp
[Test]
public async Task ShouldRequireValidXId()
{
    await Should.ThrowAsync<NotFoundException>(() =>
        TestApp.SendAsync(new DeleteXCommand(99)));
}

[Test]
public async Task ShouldDeleteX()
{
    // seed → delete → FindAsync returns null
    var deleted = await TestApp.FindAsync<X>(id);
    deleted.ShouldBeNull();
}
```

**Authorization (when `[Authorize]` is present):**
```csharp
[Test]
public async Task ShouldDenyAnonymousUser()
{
    var action = () => TestApp.SendAsync(new CreateXCommand());
    await Should.ThrowAsync<UnauthorizedAccessException>(action);
}

// If role-restricted:
[Test]
public async Task ShouldDenyNonAdministrator()
{
    await TestApp.RunAsDefaultUserAsync();
    await Should.ThrowAsync<ForbiddenAccessException>(() => TestApp.SendAsync(command));
}

[Test]
public async Task ShouldAllowAdministrator()
{
    await TestApp.RunAsAdministratorAsync();
    await Should.NotThrowAsync(() => TestApp.SendAsync(command)); // or assert result
}

// Verify the [Authorize] attribute is present on the command:
[Test]
public async Task ShouldRequireAuthorization()
{
    command.GetType().ShouldSatisfyAllConditions(
        type => type.ShouldBeDecoratedWith<AuthorizeAttribute>()
    );
}
```

**Queries:**
```csharp
[Test]
public async Task ShouldReturnExpectedResults()
{
    await TestApp.RunAsDefaultUserAsync();
    // seed data → send query → assert results
}

// If [Authorize] on query:
[Test]
public async Task ShouldDenyAnonymousUser()
{
    await Should.ThrowAsync<UnauthorizedAccessException>(() => TestApp.SendAsync(query));
}
```

### Naming convention
- `Should{ExpectedBehavior}` — e.g., `ShouldCreateTodoItem`
- `Should{ExpectedBehavior}When{Condition}` — e.g., `ShouldThrowWhenTitleTooLong`

---

## Step 4: Generate Unit Tests (when applicable)

Only generate unit tests if the module's validators contain:
- Async rules that query the database
- Custom validation logic beyond simple `NotEmpty()` / `MaxLength()`

File location:
```
tests/Application.UnitTests/{ModuleName}/Commands/{CommandName}ValidatorTests.cs
```

For simple validators (only NotEmpty, MaxLength, etc.), skip unit tests — the functional tests already cover them implicitly.

---

## Step 5: Build and Auto-Fix

After writing all files, run:

```bash
dotnet build tests/Application.FunctionalTests
```

If there are errors:
1. Read the error output carefully
2. Fix the specific files with compilation issues
3. Re-run `dotnet build` — repeat until clean

If the Unit Tests project also has new files:
```bash
dotnet build tests/Application.UnitTests
```

**Do NOT modify** existing test infrastructure files:
- `TestBase`, `TestApp`, `WebApiFactory`, `FunctionalTestSetup`, `DatabaseResetter`

---

## Step 6: Report to the User

After a successful build, summarize:

```
✅ Tests generated for module: {ModuleName}

Functional Tests:
  - tests/Application.FunctionalTests/{ModuleName}/Commands/Create{X}Tests.cs  ({n} tests)
  - tests/Application.FunctionalTests/{ModuleName}/Commands/Update{X}Tests.cs  ({n} tests)
  - tests/Application.FunctionalTests/{ModuleName}/Commands/Delete{X}Tests.cs  ({n} tests)
  - tests/Application.FunctionalTests/{ModuleName}/Queries/Get{X}Tests.cs      ({n} tests)

Unit Tests:
  - (skipped — no complex validator logic found)  OR  path + count

Build: ✅ 0 errors
```

If a DbSet was missing from `IApplicationDbContext`, warn:
```
⚠️  DbSet<{Entity}> not found in IApplicationDbContext. Tests were generated but
    FindAsync/AddAsync calls may fail at runtime. Add the DbSet to fix this.
```
