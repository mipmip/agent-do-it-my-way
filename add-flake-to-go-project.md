# Adding Nix Flake Support to Go Projects

A step-by-step guide for adding Nix flake support to Go projects, resulting in a clean PR ready for merge.

## Prerequisites

- Working Nix installation with flakes enabled
- Go project with `go.mod` file
- Git repository

## Step 1: Explore the Project Structure

Before creating the flake, understand the project:

```bash
# Read the README to understand the project
cat README.md

# Check Go module and version
cat go.mod

# Identify the main package
cat main.go

# Check for submodules or special directories
ls -la
```

**Key things to identify:**
- Project purpose and features
- Current version (check git tags, releases, or documentation)
- Build requirements
- Separate Go modules (like `lambda/go.mod` in updo)
- Existing version injection patterns (check for ldflags usage)

## Step 2: Create flake.nix

Create a `flake.nix` with the following structure:

```nix
{
  description = "Brief description of the project";

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
          default = pkgs.buildGoModule rec {
            pname = "project-name";
            version = "X.Y.Z";  # Use latest release version

            src = ./.;

            vendorHash = null;  # Will fix in next step

            # Exclude separate Go modules if any
            # excludedPackages = [ "./submodule" ];

            ldflags = [
              "-s"
              "-w"
              # Add version injection if the project uses it
              # Check main.go for version variables
              # "-X main.version=${version}"
              # "-X main.commit=${src.rev or "unknown"}"
              # "-X main.date=1970-01-01T00:00:00Z"
            ];

            meta = with pkgs.lib; {
              description = "Command-line tool description";
              homepage = "https://github.com/owner/repo";
              license = licenses.mit;  # Adjust based on LICENSE file
              maintainers = [ ];
              mainProgram = "binary-name";
            };
          };

          project-name = self.packages.${system}.default;
        }
      );

      apps = forAllSystems (system: {
        default = {
          type = "app";
          program = "${self.packages.${system}.default}/bin/binary-name";
        };

        project-name = self.apps.${system}.default;
      });

      devShells = forAllSystems (system:
        let
          pkgs = nixpkgs.legacyPackages.${system};
        in
        {
          default = pkgs.mkShell {
            buildInputs = with pkgs; [
              go
              gopls
              gotools
              go-tools
            ];

            shellHook = ''
              echo "Project development environment"
              echo "Go version: $(go version)"
            '';
          };
        }
      );
    };
}
```

## Step 3: Stage flake.nix and Generate flake.lock

Nix requires files to be tracked by git:

```bash
git add flake.nix
nix flake update
```

This creates `flake.lock` with pinned dependencies.

## Step 4: Fix vendorHash

The `vendorHash` ensures reproducible builds of Go dependencies:

```bash
# This will fail but show the correct hash
nix build --no-link

# Copy the hash from error message:
# "got: sha256-XXXXX..."

# Update flake.nix with the correct vendorHash
# Then rebuild to verify
nix build --no-link
```

## Step 5: Handle Build Errors

### Separate Go Modules

If build fails with "does not contain package":

```bash
# Check for separate go.mod files
find . -name go.mod

# If found (e.g., lambda/go.mod), add to flake.nix:
excludedPackages = [ "./lambda" ];
```

### Version Injection

Check if the project uses version variables:

```bash
# Look for version variables in main.go
grep -n "version\|commit\|date" main.go

# Check build configuration
cat .goreleaser.yaml 2>/dev/null | grep ldflags
```

If found, add to `ldflags` in flake.nix to match the build system.

## Step 6: Verify the Build

Test the flake thoroughly:

```bash
# Build successfully
nix build --no-link

# Test running the binary
nix run . -- --version
nix run . -- --help

# Verify flake structure
nix flake check

# Test development shell
nix develop -c go version
```

## Step 7: Update Documentation

### README.md

Add a Nix installation section after other package managers:

```markdown
<details>
<summary>Nix/NixOS</summary>

**Run directly:**

```bash
nix run github:owner/repo -- [args]
```

**Temporary shell:**

```bash
nix shell github:owner/repo
project-name --version
```

**System flake integration:**

```nix
{
  inputs.project-name.url = "github:owner/repo";

  outputs = { self, nixpkgs, project-name }: {
    nixosConfigurations.myhost = nixpkgs.lib.nixosSystem {
      modules = [{
        environment.systemPackages = [ project-name.packages.x86_64-linux.default ];
      }];
    };
  };
}
```

</details>
```

### docs/RELEASE.md or CONTRIBUTING.md

Add a section on maintaining the flake after releases:

```markdown
## Updating the Nix Flake

After creating a release, update the Nix flake:

1. **Update the version in flake.nix**

   ```bash
   # Edit flake.nix and update the version field
   # Change: version = "X.Y.Z";
   # To:     version = "X.Y.Z+1";
   ```

2. **Update vendorHash if dependencies changed**

   If `go.mod` or `go.sum` changed:

   ```bash
   # Build will fail and show the correct hash
   nix build --no-link

   # Update vendorHash in flake.nix with the hash from error
   ```

3. **Test the flake**

   ```bash
   # Update flake.lock
   nix flake update

   # Test the version is correct
   nix run . -- --version
   ```

4. **Commit and push**

   ```bash
   git add flake.nix flake.lock
   git commit -m "chore: update nix flake to vX.Y.Z"
   git push origin main
   ```
```

## Step 8: Prepare for PR

```bash
# Stage all files
git add flake.nix flake.lock

# Check what will be committed
git status

# Review changes
git diff --cached
```

## Step 9: Commit Message Template

```
feat: add nix flake for declarative installation

Add flake.nix and flake.lock to enable Nix users to install and run <project>
through the Nix package manager. The flake provides:

- buildGoModule derivation with proper Go dependencies (vX.Y.Z)
- Default package and app outputs for easy installation
- Development shell with Go tooling (go, gopls, gotools)
- Support for all Nix-supported systems (x86_64/aarch64 linux/darwin)
- [Version injection via ldflags matching build system behavior]
- Native Nix implementation without external dependencies

Documentation updates:
- Added Nix/NixOS installation section to README with usage examples
- Added flake maintenance instructions to docs/RELEASE.md

Nix users can now install <project> with:
  nix profile install github:owner/repo

Run directly without installing:
  nix run github:owner/repo

Or integrate into NixOS system configurations.
```

## Troubleshooting

### "Path 'flake.nix' is not tracked by Git"
```bash
git add flake.nix
```

### "hash mismatch in fixed-output derivation"
Update `vendorHash` with the hash from the error message.

### Build fails with "does not contain package"
Check for separate Go modules and add to `excludedPackages`.

### Version shows as "dev"
Add ldflags for version injection (see Step 5).

### "dirty Git tree" warnings
These are normal during development; they'll disappear after committing.

## Best Practices

1. **Use native Nix** - Avoid `flake-utils` dependency for simpler flakes
2. **Match existing patterns** - If the project uses goreleaser, mirror its ldflags
3. **Test all systems** - Use `nix flake check --all-systems` if possible
4. **Document maintenance** - Help maintainers update the flake after releases
5. **Keep it minimal** - Only add what's necessary for the package to work

## Verification Checklist

Before creating the PR:

- [ ] `nix build --no-link` succeeds
- [ ] `nix run . -- --version` shows correct version
- [ ] `nix flake check` passes
- [ ] `nix develop` provides working Go environment
- [ ] README has Nix installation section
- [ ] Release/contributing docs updated with flake maintenance
- [ ] All files staged: `git status`
- [ ] Commit message follows project conventions

---

This guide was created from a real implementation on the [updo](https://github.com/Owloops/updo) project and should work for most Go CLI tools.
