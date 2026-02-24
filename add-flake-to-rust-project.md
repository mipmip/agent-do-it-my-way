# Adding Nix Flake Support to Rust Projects

A step-by-step guide for adding Nix flake support to Rust projects, resulting in a clean PR ready for merge.

## Prerequisites

- Working Nix installation with flakes enabled
- Rust project with `Cargo.toml` file
- Git repository

## Project Investigation Phase

Begin by understanding your project's structure and dependencies. Examine the README, Cargo.toml configuration, and main entry point. Check for any workspace configurations or multiple crates in subdirectories.

## Creating the Flake Configuration

The `flake.nix` file forms the foundation. It should define:

**Basic structure components:**
- Description and nixpkgs input reference
- Multi-system output support using native Nix (x86_64-linux, aarch64-linux, x86_64-darwin, aarch64-darwin)
- `rustPlatform.buildRustPackage` derivation with package metadata
- Development shell providing Rust tooling (cargo, rustc, rust-analyzer, rustfmt, clippy)
- App outputs for direct execution

**Key configuration elements:**
- Project name (pname) matching the binary name
- Current version matching latest release
- Source set to current directory (`src = ./.`)
- `cargoHash` for reproducible dependency verification
- License and homepage metadata
- `mainProgram` set to the binary name

**Example flake.nix structure:**

```nix
{
  description = "Project description";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
  };

  outputs = { self, nixpkgs }:
    let
      systems = [ "x86_64-linux" "aarch64-linux" "x86_64-darwin" "aarch64-darwin" ];
      forAllSystems = nixpkgs.lib.genAttrs systems;
    in
    {
      packages = forAllSystems (system:
        let
          pkgs = nixpkgs.legacyPackages.${system};
        in
        {
          default = pkgs.rustPlatform.buildRustPackage {
            pname = "project-name";
            version = "0.1.0";

            src = ./.;

            cargoHash = ""; # Will be filled after first build attempt

            meta = with pkgs.lib; {
              description = "Project description";
              homepage = "https://github.com/owner/repo";
              license = licenses.mit;
              mainProgram = "binary-name";
            };
          };
        }
      );

      apps = forAllSystems (system: {
        default = {
          type = "app";
          program = "${self.packages.${system}.default}/bin/binary-name";
        };
      });

      devShells = forAllSystems (system:
        let
          pkgs = nixpkgs.legacyPackages.${system};
        in
        {
          default = pkgs.mkShell {
            buildInputs = with pkgs; [
              cargo
              rustc
              rust-analyzer
              rustfmt
              clippy
            ];

            RUST_SRC_PATH = "${pkgs.rust.packages.stable.rustPlatform.rustLibSrc}";
          };
        }
      );
    };
}
```

## Git Integration and Dependency Locking

Nix requires files to be tracked by Git before processing. Stage the `flake.nix` file and run `nix flake update` to generate `flake.lock`, which pins all dependencies.

```bash
git add flake.nix
nix flake update
git add flake.lock
```

## Cargo Hash Resolution

The `cargoHash` ensures reproducible builds. Run `nix build --no-link` to trigger a build attempt. The error message will reveal the correct hash—copy it and update the configuration, then rebuild to verify.

```bash
nix build --no-link 2>&1 | grep "got:"
```

The output will show something like:
```
got:    sha256-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=
```

Copy this hash and update the `cargoHash` field in `flake.nix`.

## Handling Complex Projects

**For workspace projects:** If your project uses Cargo workspaces with multiple crates, the flake will automatically handle all workspace members. No special configuration needed.

**For binary selection:** If you have multiple binaries defined in `Cargo.toml`, ensure `mainProgram` points to the primary binary you want users to run.

## Verification Process

Test thoroughly before proceeding:
- `nix build --no-link` confirms successful compilation
- `nix run . -- --version` validates the binary runs correctly
- `nix flake check` verifies flake structure
- `nix develop -c cargo --version` confirms development shell functionality

```bash
# Full verification sequence
nix build --no-link
nix run . -- --version
nix flake check
nix develop -c cargo --version
```

## Documentation Updates

**README additions:** Include a Nix/NixOS section showing installation methods:

```markdown
#### Nix/NixOS

For users with Nix installed, you can use [ProjectName] without installing it:

```bash
# Run directly
nix run github:owner/repo

# Try it in a temporary shell
nix shell github:owner/repo
project-name --version

# Add to your NixOS configuration or home-manager
{
  environment.systemPackages = [
    inputs.project-name.packages.${system}.default
  ];
}
```

To build from source with Nix:

```bash
nix build
nix run . -- --version
```

For development:

```bash
nix develop
cargo build
```

For maintainers updating the flake, see [docs/maintaining-nix-flake.md](docs/maintaining-nix-flake.md).
```

**Maintenance documentation:** Create a `docs/maintaining-nix-flake.md` file explaining how to:
- Update the version in `flake.nix`
- Recalculate `cargoHash` when dependencies change
- Test before committing
- Update `flake.lock` periodically

Example maintenance documentation structure:

```markdown
# Maintaining the Nix Flake

## Release Process

When releasing a new version:

1. Update the `version` field in `flake.nix`
2. If dependencies changed, recalculate `cargoHash`:
   ```bash
   git add flake.nix Cargo.toml Cargo.lock
   # Clear the hash temporarily
   sed -i 's/cargoHash = ".*"/cargoHash = ""/' flake.nix
   # Get the correct hash
   nix build --no-link 2>&1 | grep "got:"
   # Update flake.nix with the new hash
   ```
3. Test the build:
   ```bash
   nix build --no-link
   nix run . -- --version
   nix flake check
   ```
4. Update flake.lock: `nix flake update`
5. Commit all changes together
```

## Preparing for PR

```bash
# Stage all files
git add flake.nix flake.lock

# Check what will be committed
git status

# Review changes
git diff --cached
```

## Commit Message Template

```
feat: add nix flake for declarative installation

Add flake.nix and flake.lock to enable Nix users to install and run <project>
through the Nix package manager. The flake provides:

- rustPlatform.buildRustPackage derivation with Cargo integration (vX.Y.Z)
- Default package and app outputs for easy installation
- Development shell with Rust tooling (cargo, rustc, rust-analyzer, clippy)
- Support for all Nix-supported systems (x86_64/aarch64 linux/darwin)
- Native Nix implementation without external dependencies

Documentation updates:
- Added Nix/NixOS installation section to README with usage examples
- Added flake maintenance instructions to docs/maintaining-nix-flake.md

Nix users can now install <project> with:
  nix profile install github:owner/repo

Run directly without installing:
  nix run github:owner/repo

Or integrate into NixOS system configurations.
```

## Verification Checklist

Before creating the PR:

- [ ] `nix build --no-link` succeeds
- [ ] `nix run . -- --version` shows correct version
- [ ] `nix flake check` passes
- [ ] `nix develop` provides working Rust environment
- [ ] README has Nix installation section
- [ ] Maintenance documentation created
- [ ] All files staged: `git status`
- [ ] Commit message follows project conventions

## Troubleshooting Common Issues

- **Untracked files:** Always stage `flake.nix` with `git add` before building
- **Hash mismatches:** Copy the correct hash from error output (look for "got:" line)
- **Workspace issues:** Nix handles Cargo workspaces automatically, no special config needed
- **Wrong binary:** Ensure `mainProgram` matches your primary binary name in Cargo.toml
- **Git warnings:** "dirty tree" warnings disappear after committing staged changes
- **Development shell issues:** Verify RUST_SRC_PATH is set correctly for rust-analyzer

## Best Practices

1. **Use native Nix** - Avoid `flake-utils` dependency for simpler flakes
2. **Match existing patterns** - Mirror build configuration from Cargo.toml
3. **Test all systems** - Use `nix flake check --all-systems` if possible
4. **Document maintenance** - Help maintainers update the flake after releases
5. **Keep it minimal** - Only add what's necessary for the package to work
