# Setup Issues

## 1. Missing `using Xunit;` Directive

**Symptom:** `dotnet test` produced build errors referencing `IClassFixture<>`, `FactAttribute`, and `Fact`.

**Investigation:** Searched for `IClassFixture<>` and confirmed it belongs to the xUnit library. The xUnit package references were already present in the `.csproj` file, so the issue was not a missing package but a missing using directive.

**Resolution:** Added the following to the top of `TaskApiTests.cs`:

```csharp
using Xunit;
```

## 2. Missing `FluentAssertions.dll` at Runtime

**Symptom:** After resolving the build errors above, `dotnet test` produced a new error:

```
Testhost process for source(s) '...\SupportEngineerChallenge.Tests.dll' exited with error: Error:
  An assembly specified in the application dependencies manifest (SupportEngineerChallenge.Tests.deps.json) was not found:
    package: 'FluentAssertions', version: '6.12.0'
    path: 'lib/net6.0/FluentAssertions.dll'
```

**Investigation:**

- Checked `bin/Debug/net8.0` and found that `FluentAssertions.dll` was missing despite `dotnet restore` completing successfully.
- Checked `SupportEngineerChallenge.Tests.csproj` and the FluentAssertions package reference appeared normal.
- Ran a detailed build log:

```bash
dotnet build tests/SupportEngineerChallenge.Tests/SupportEngineerChallenge.Tests.csproj -v detailed > build_log.txt 2>&1
```

- Found the following in the log for FluentAssertions:

```
This reference is not "CopyLocal" because at least one source item had "Private" set to "false"
and no source items had "Private" set to "true".
```

- However, this same message appeared for every project reference, suggesting it was a broader build configuration issue rather than something specific to FluentAssertions.
- Installed and opened the MSBuild Structured Log Viewer for further analysis, but it did not surface any additional insights beyond what was already concluded.

**Pivot:** Shifted approach from build log analysis to reviewing xUnit documentation directly. The xUnit v2 Getting Started page showed a template `.csproj` with three required package references:

1. `xunit` which was already present
2. `xunit.runner.visualstudio` which was already present
3. `Microsoft.NET.Test.Sdk` which was **missing**

The documentation also explicitly stated:

> `xunit.runner.visualstudio` and `Microsoft.NET.Test.Sdk` are used to enable support for VSTest-based runners, like `dotnet test` and Visual Studio Test Explorer.

**Resolution:** Added the missing package:

```bash
dotnet add package Microsoft.NET.Test.Sdk
```

After adding this package, `dotnet test` executed successfully and all tests passed.
