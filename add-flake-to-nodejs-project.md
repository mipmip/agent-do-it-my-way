# Adding Nix Flake Support to Node.js Projects

A step-by-step guide for adding Nix flake support to Node.js/TypeScript projects, resulting in a clean PR ready for merge.

## Prerequisites

- Working Nix installation with flakes enabled
- Node.js project with `package.json` file
- Git repository

## Step 1: Explore the Project Structure

Before creating the flake, understand the project:

```bash
# Read the README to understand the project
cat README.md

# Check package.json for project details
cat package.json

# Identify the entry point and bin configuration
grep -A5 '"bin"' package.json
grep -A5 '"main"' package.json

# Check Node.js version requirements
grep -A2 '"engines"' package.json

# Check for TypeScript configuration
ls tsconfig.json 2>/dev/null && cat tsconfig.json
```

**Key things to identify:**
- Project purpose and features
- Current version (from package.json)
- Build requirements (TypeScript, bundlers, etc.)
- Binary entry point (bin field in package.json)
- Node.js version requirements (engines field)
- Build scripts (build, compile, etc.)

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
          default = pkgs.buildNpmPackage {
            pname = "project-name";
            version = "X.Y.Z";  # Use version from package.json

            src = ./.;

            npmDepsHash = "";  # Will be filled after first build attempt

            # Build phase (for TypeScript projects)
            buildPhase = ''
              runHook preBuild
              npm run build
              runHook postBuild
            '';

            # Install the built package
            installPhase = ''
              runHook preInstall
              mkdir -p $out/lib/node_modules/project-name
              cp -r dist package.json $out/lib/node_modules/project-name/

              # Copy node_modules for runtime dependencies
              cp -r node_modules $out/lib/node_modules/project-name/

              # Create bin wrapper
              mkdir -p $out/bin
              cat > $out/bin/project-name << 'EOF'
              #!/usr/bin/env node
              require('../lib/node_modules/project-name/dist/cli/index.js');
              EOF
              chmod +x $out/bin/project-name

              # Make it a proper Node.js script
              substituteInPlace $out/bin/project-name \
                --replace '#!/usr/bin/env node' '#!${pkgs.nodejs}/bin/node'

              runHook postInstall
            '';

            meta = with pkgs.lib; {
              description = "Command-line tool description";
              homepage = "https://github.com/owner/repo";
              license = licenses.mit;  # Adjust based on LICENSE file
              maintainers = [ ];
              mainProgram = "binary-name";
              platforms = platforms.all;
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
              nodejs_22  # Match project's Node.js version requirement
              nodePackages.npm
              nodePackages.typescript
              nodePackages.typescript-language-server
            ];

            shellHook = ''
              echo "Project development environment"
              echo "Node.js version: $(node --version)"
              echo "npm version: $(npm --version)"
            '';
          };
        }
      );
    };
}
```

### Build Configuration Variants

**For projects with custom build output directories:**

Adjust the `installPhase` to match the actual output directory:

```nix
installPhase = ''
  runHook preInstall
  mkdir -p $out/lib/node_modules/project-name
  cp -r build package.json $out/lib/node_modules/project-name/  # 'build' instead of 'dist'
  # ... rest of install phase
'';
```

**For projects without a build step (pure JavaScript):**

Remove the `buildPhase` and copy source directly:

```nix
buildPhase = ''
  runHook preBuild
  # No build needed for pure JS projects
  runHook postBuild
'';

installPhase = ''
  runHook preInstall
  mkdir -p $out/lib/node_modules/project-name
  cp -r src package.json $out/lib/node_modules/project-name/
  cp -r node_modules $out/lib/node_modules/project-name/
  # ... create bin wrapper
  runHook postInstall
'';
```

**For monorepo projects:**

If the project uses workspaces, you may need to build specific packages:

```nix
buildPhase = ''
  runHook preBuild
  npm run build --workspace=packages/cli
  runHook postBuild
'';
```

## Step 3: Stage flake.nix and Generate flake.lock

Nix requires files to be tracked by git:

```bash
git add flake.nix
nix flake update
```

This creates `flake.lock` with pinned dependencies.

## Step 4: Fix npmDepsHash

The `npmDepsHash` ensures reproducible builds of npm dependencies:

```bash
# This will fail but show the correct hash
nix build --no-link

# Copy the hash from error message:
# "got: sha256-XXXXX..."

# Update flake.nix with the correct npmDepsHash
# Then rebuild to verify
nix build --no-link
```

## Step 5: Handle Build Errors

### Missing Build Script

If build fails with "missing script: build":

```bash
# Check available scripts
cat package.json | grep -A20 '"scripts"'

# Adjust buildPhase to use the correct script, e.g.:
# npm run compile
# npm run bundle
# npx tsc
```

### Binary Path Issues

If the binary doesn't work, check the entry point:

```bash
# Check bin configuration in package.json
cat package.json | grep -A5 '"bin"'

# Verify the built output exists
ls -la dist/  # or build/, lib/, etc.
```

Update the `installPhase` to match the actual binary location.

### Native Dependencies

If the project uses native Node.js addons (e.g., node-gyp):

```nix
buildInputs = with pkgs; [
  nodejs
  python3  # Often needed for node-gyp
];

nativeBuildInputs = with pkgs; [
  pkg-config
  # Add other native dependencies as needed
];
```

### Optional Dependencies

If build fails on optional dependencies:

```nix
npmFlags = [ "--ignore-scripts" ];  # Skip postinstall scripts if problematic
```

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
nix develop -c node --version
nix develop -c npm --version
```

## Step 7: Update Documentation

### README.md

Add a Nix installation section after other installation methods:

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

**Development:**

```bash
git clone https://github.com/owner/repo
cd repo
nix develop
npm install
npm run dev
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

2. **Update npmDepsHash if dependencies changed**

   If `package.json` or `package-lock.json` changed:

   ```bash
   # Clear the hash temporarily (set to empty string)
   # Build will fail and show the correct hash
   nix build --no-link

   # Update npmDepsHash in flake.nix with the hash from error
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

- buildNpmPackage derivation with npm dependency vendoring (vX.Y.Z)
- Default package and app outputs for easy installation
- Development shell with Node.js tooling (node, npm, typescript, typescript-language-server)
- Support for all Nix-supported systems (x86_64/aarch64 linux/darwin)
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
Update `npmDepsHash` with the hash from the error message.

### "missing script: build"
Check the scripts in package.json and update buildPhase accordingly.

### Binary not found in $out/bin
Verify the installPhase creates the bin wrapper correctly and points to the right entry file.

### "Cannot find module" at runtime
Ensure node_modules is copied to the output directory in installPhase.

### TypeScript compilation errors
The project may need specific TypeScript version. Add to devShells or ensure tsconfig.json is compatible.

### "dirty Git tree" warnings
These are normal during development; they'll disappear after committing.

## Best Practices

1. **Use native Nix** - Avoid `flake-utils` dependency for simpler flakes
2. **Match Node.js version** - Use the same major version as specified in package.json engines
3. **Test all commands** - Verify --version, --help, and actual functionality
4. **Document maintenance** - Help maintainers update the flake after releases
5. **Keep it minimal** - Only add what's necessary for the package to work
6. **Pin nixpkgs** - flake.lock ensures reproducible builds

## Verification Checklist

Before creating the PR:

- [ ] `nix build --no-link` succeeds
- [ ] `nix run . -- --version` shows correct version
- [ ] `nix run . -- --help` displays help
- [ ] `nix flake check` passes (warnings about meta in apps are OK)
- [ ] `nix develop` provides working Node.js environment
- [ ] README has Nix installation section
- [ ] Release/contributing docs updated with flake maintenance
- [ ] All files staged: `git status`
- [ ] Commit message follows project conventions

---

This guide was created from a real implementation on the [spec-gen](https://github.com/clay-good/spec-gen) project and should work for most Node.js/TypeScript CLI tools.
