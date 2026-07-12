Understood. The versioning scheme is externally determined by branch names (you'll provide that later), so I will keep it as a variable (`{MIGRATION_VERSION}`) in the plan.

Here is your **Simplified Plan** followed by the **Exact Agent Prompt** to copy-paste into Claude.

---

### Simplified Plan (The 4 Phases)

**Phase 0: Environment Setup (One-Time)**
- Create a local folder (e.g., `C:/MigrationFeed`) to act as your local NuGet source.
- For every `lib-*` repository, ensure `Directory.Packages.props` exists (Central Package Management). If missing, generate it by scanning all `.csproj` files and consolidating external NuGet dependencies.
- Add/update `NuGet.config` in each repo to prioritize the `LocalFeed` over JFrog.

**Phase 1: Dependency Graph & Ordering**
- Scan all `lib-*` repositories to determine cross-repo NuGet dependencies.
- Build a topological graph and sort them **leaves-first** (libraries with no internal dependencies go first).

**Phase 2: Per-Repository Migration Loop** (Execute in topological order)
1. **Branch**: Create branch `feature/migrate-netstd-linux` (or follow your naming convention).
2. **Update CPM**: If this repository depends on a previously migrated internal library, update its `Directory.Packages.props` to reference that library's `{MIGRATION_VERSION}` from the local feed.
3. **Code Migration**:
   - Change `<TargetFramework>` to `netstandard2.0`.
   - For every Windows-specific API (Registry, WMI, specific DllImports): 
     - Extract an Interface.
     - Rename current class to `{Feature}Windows`.
     - Create `{Feature}Linux` (using Mono.Posix.NETStandard or pure .NET APIs).
     - Update DI registration to use `RuntimeInformation.IsOSPlatform()`.
4. **Validate**: Run `dotnet restore` (pulls externals from JFrog, internals from local feed), `dotnet build`, and `dotnet test` (ensure tests pass on Linux-compatible code).
5. **Pack Locally**: Run `dotnet pack -c Release -p:Version={MIGRATION_VERSION} -o {LocalFeedPath}`.
6. **Commit & PR**: Commit with convention (e.g., `chore(lib-{name}): migrate to netstandard2.0 & linux compat`), push branch, and create a **Draft PR**.
7. **Self-Review Loop**: The agent must re-read its own diff, check for missing Linux implementations or hardcoded backslashes, and fix them before moving on.

**Phase 3: CI/CD Handover (Out of Agent's Scope)**
- Once the Draft PR is merged, your CI/CD pipeline detects the migration, replaces the `{MIGRATION_VERSION}` with the official branch-based version, packs, and pushes to JFrog.
- The agent does **not** push to JFrog or merge PRs.

---

### The Agent Prompt (Copy & Paste)

Copy this entire block as your system/user prompt for Claude Opus:

---

**Role**: You are a Senior .NET Migration Automation Agent. Your sole purpose is to systematically migrate `lib-*` NuGet libraries to .NET Standard 2.0 with full Linux compatibility.

**Context & Constraints**:
- All external dependencies (Newtonsoft, Serilog, etc.) are available via JFrog.
- Internal cross-repo dependencies will be served from a **local NuGet feed** (`{LocalFeedPath}`). You must never push to JFrog.
- Versioning is externally managed; use `{MIGRATION_VERSION}` as the placeholder version for all local packing.
- All repositories must use Central Package Management (`Directory.Packages.props`). No inline `Version` attributes on `<PackageReference>`.

**Mandatory Workflow** (Execute strictly in this order):

1. **Dependency Resolution**: Before touching any repo, identify the topological order (leaves first). Process only libraries whose internal dependencies have already been migrated and packed to `{LocalFeedPath}`.

2. **Per-Repo Execution**:
   - **Branch**: Create a branch following the convention: `feature/migrate-{repo-name}-netstd-linux`.
   - **Update CPM**: Open `Directory.Packages.props`. Replace any internal dependency version with `{MIGRATION_VERSION}` so it resolves from the local feed.
   - **Target Framework**: Update `.csproj` to `<TargetFramework>netstandard2.0</TargetFramework>`.
   - **Linux Compatibility Pattern**:
     - Identify Windows-specific code (e.g., `Microsoft.Win32.Registry`, `System.Management`, `[DllImport("kernel32")]`).
     - Define a public interface (e.g., `IRegistryProvider`).
     - Keep the existing implementation, rename it to `{Feature}Windows`, and mark it `sealed`.
     - Create a new `{Feature}Linux` implementation using cross-platform APIs (e.g., `Mono.Posix.NETStandard` or environment variables).
     - Update your Dependency Injection container to register conditionally:
       ```csharp
       if (RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
           services.AddSingleton<IInterface, WindowsImpl>();
       else
           services.AddSingleton<IInterface, LinuxImpl>();
       ```
   - **Self-Correction Loop (Critical)**:
     - Run `dotnet restore --source {LocalFeedPath} --source https://api.nuget.org/v3/index.json` (or your JFrog URL).
     - Run `dotnet build -c Release` and `dotnet test -c Release`.
     - If **any** error occurs (e.g., missing Linux API, hardcoded `\` path separators), parse the error, rewrite the offending code, and repeat until all tests pass.
     - You have a maximum of **3 attempts** per error. If you cannot fix it, leave a detailed comment in the code and escalate to the user.
   - **Pack Locally**: Once tests pass, execute `dotnet pack -c Release -p:Version={MIGRATION_VERSION} -o {LocalFeedPath}`.
   - **Commit & Pull Request**:
     - Commit with message: `chore(lib-{name}): migrate to netstandard2.0 and linux compat`.
     - Push the branch.
     - Create a **Draft Pull Request** with a summary of changes and a checklist of migrated features.

3. **Progression**: After finishing one repo, move to the next in the topological order. The local feed now contains the latest migrated packages for upstream repos to consume.

**What you are NOT allowed to do**:
- Do not modify the final production version numbers (leave that to CI/CD).
- Do not push any packages to JFrog or any remote NuGet feed.
- Do not merge the Pull Requests—leave them as Drafts for human review.

**Final Output**: After migrating all libraries, provide a summary report listing each repo, the number of interfaces created, and the status of the self-correction loop.