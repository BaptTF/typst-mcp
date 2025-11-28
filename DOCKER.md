# Docker Build System

This project uses a multi-stage Docker build system with a separate base image containing pre-compiled Typst documentation. This approach significantly speeds up builds by avoiding the need to recompile Typst documentation on every build.

## Architecture

### Base Image (`Dockerfile.base`)
- Contains pre-compiled Typst documentation
- Built from Rust nightly image
- Only needs to be rebuilt when Typst documentation changes
- Tagged as `ghcr.io/johannesbrandenburger/typst-mcp-base:latest`

### Runtime Image (`Dockerfile.runtime`)
- Contains the actual MCP server
- Uses the base image for Typst documentation
- Built from Python slim image
- Includes typst binary, Python dependencies, and application code
- Tagged as `ghcr.io/johannesbrandenburger/typst-mcp:latest`

## CI/CD System

### GitHub Actions Workflow

The CI/CD system uses a sophisticated two-stage build process with intelligent caching and dependency management.

#### Workflow Triggers
- **Push to main/master**: Triggers both base and runtime builds
- **Pull requests**: Builds both images for testing (no push)
- **Release**: Full build and push with release tags
- **Manual dispatch**: Can trigger builds manually

#### Base Image Build Logic

The base image follows this decision tree:

```yaml
# Build base image if ANY of these conditions are true:
- Dockerfile.base was modified
- Manual workflow dispatch
- Creating a release
- Base image doesn't exist in registry yet (first-time setup)
```

**Key Features:**
- **Existence Check**: Checks if `ghcr.io/johannesbrandenburger/typst-mcp-base:latest` already exists
- **Smart Rebuilding**: Only rebuilds when necessary, not on every run
- **First-time Setup**: Automatically builds base image if it doesn't exist
- **Change Detection**: Monitors `Dockerfile.base` for changes

#### Runtime Image Build Logic

The runtime image:
- **Always builds** (for code changes)
- **Depends on base image** completion
- **Uses cached base image** when available
- **Builds quickly** (~2 minutes) when base exists

#### Build Process Flow

```
1. Push to main branch
   ↓
2. build-base job starts
   ├─ Check if base image exists in registry
   ├─ If exists + no changes → SKIP
   └─ If doesn't exist OR changed → BUILD
   ↓
3. build-runtime job starts (waits for base)
   ├─ Extract metadata for proper tagging
   ├─ Build runtime image using base
   └─ Push to registry
```

### Automated Builds (GitHub Actions)
- **Base image**: Built only when needed (changes or first-time)
- **Runtime image**: Built on every push/release, uses cached base image
- **Smart Dependencies**: Runtime waits for base completion
- **Registry Management**: Automatic tagging and pushing

### Local Development

Use the provided build script:

```bash
# Build both images
./build.sh

# Build only base image (when docs change)
./build.sh base

# Build only runtime image (for code changes)
./build.sh runtime

# Push images to registry
./build.sh push

# Run the container
./build.sh run
```

### Manual Docker Commands

```bash
# Build base image
docker build -f Dockerfile.base -t ghcr.io/johannesbrandenburger/typst-mcp-base:latest .

# Build runtime image
docker build -f Dockerfile.runtime -t ghcr.io/johannesbrandenburger/typst-mcp:latest .

# Push both images
docker push ghcr.io/johannesbrandenburger/typst-mcp-base:latest
docker push ghcr.io/johannesbrandenburger/typst-mcp:latest
```

## Benefits

1. **Faster Builds**: Runtime image builds in ~2 minutes instead of ~15 minutes
2. **Better Caching**: Base image is reused across multiple runtime builds
3. **Selective Rebuilding**: Only rebuild documentation when Typst updates
4. **Smaller Transfer**: Only runtime layer changes need to be pushed/pulled
5. **Development Efficiency**: Quick iteration on code changes

## Base Image Management

### Creating the Base Image

#### First-Time Setup
The base image is automatically created on the first CI/CD run. No manual intervention needed.

#### Manual Recreation
If you need to recreate the base image manually:

```bash
# Method 1: Using build script
./build.sh base

# Method 2: Direct Docker command
docker build -f Dockerfile.base -t ghcr.io/johannesbrandenburger/typst-mcp-base:latest .

# Method 3: Force rebuild (no cache)
docker build --no-cache -f Dockerfile.base -t ghcr.io/johannesbrandenburger/typst-mcp-base:latest .
```

#### Triggering Base Image Rebuild in CI/CD

**Automatic Triggers:**
- Modify `Dockerfile.base`
- Create a new release
- Manual workflow dispatch in GitHub Actions

**Manual Trigger:**
1. Go to GitHub Actions tab
2. Select "Build and Push Docker Images" workflow
3. Click "Run workflow" → "Run workflow"

#### Verifying Base Image

Check if base image exists and contains documentation:

```bash
# Check if image exists locally
docker images | grep typst-mcp-base

# Check image contents
docker run --rm ghcr.io/johannesbrandenburger/typst-mcp-base:latest ls -la /usr/local/share/typst-docs/

# Verify documentation file exists
docker run --rm ghcr.io/johannesbrandenburger/typst-mcp-base:latest ls -la /usr/local/share/typst-docs/main.json
```

### When to Rebuild Each Image

#### Base Image (`ghcr.io/johannesbrandenburger/typst-mcp-base`)
**Rebuild when:**
- Typst repository updates (new documentation features)
- Documentation generation requirements change
- Rust toolchain needs updating
- Base image becomes corrupted or missing
- Manual update of Typst documentation version

**Runtime Image (`ghcr.io/johannesbrandenburger/typst-mcp`)**
**Rebuild when:**
- Application code changes (`server.py`, etc.)
- Python dependencies update (`pyproject.toml`, `uv.lock`)
- Typst binary version updates
- System dependencies change
- Configuration files change

## Troubleshooting

### Common CI/CD Issues

#### Base Image Not Found
**Error:** `failed to solve: ghcr.io/johannesbrandenburger/typst-mcp-base:latest: not found`

**Solutions:**
1. **Wait for completion**: Base image might still be building
2. **Manual trigger**: Run workflow manually to create base image
3. **Check logs**: Verify base image job completed successfully
4. **Force rebuild**: Push a small change to `Dockerfile.base`

#### Repository Name Must Be Lowercase
**Error:** `invalid reference format: repository name must be lowercase`

**Solution:** This is automatically handled by the `docker/metadata-action`. If it persists, check that the workflow is using the correct metadata extraction.

#### Build Timeouts
**Symptoms:** Builds taking too long or timing out

**Solutions:**
1. **Check base image**: Ensure base image exists (runtime builds are fast)
2. **Network issues**: GitHub Actions may have connectivity problems
3. **Resource limits**: Check if you're exceeding GitHub Actions limits

### Local Development Issues

#### Base Image Issues
```bash
# Check if base image exists
docker images | grep typst-mcp-base

# Force rebuild base image
docker build --no-cache -f Dockerfile.base -t ghcr.io/johannesbrandenburger/typst-mcp-base:latest .

# Verify base image contents
docker run --rm ghcr.io/johannesbrandenburger/typst-mcp-base:latest ls -la /usr/local/share/typst-docs/
```

#### Runtime Image Issues
```bash
# Force rebuild runtime image
docker build --no-cache -f Dockerfile.runtime -t ghcr.io/johannesbrandenburger/typst-mcp:latest .

# Build with verbose output
docker build --progress=plain -f Dockerfile.runtime -t ghcr.io/johannesbrandenburger/typst-mcp:latest .
```

#### Documentation Problems
Check that the base image was built successfully:
```bash
# Verify documentation file exists
docker run --rm ghcr.io/johannesbrandenburger/typst-mcp-base:latest ls -la /usr/local/share/typst-docs/main.json

# Check documentation size (should be several MB)
docker run --rm ghcr.io/johannesbrandenburger/typst-mcp-base:latest du -h /usr/local/share/typst-docs/main.json
```

### Debugging CI/CD

#### Checking Build Logs
1. Go to GitHub Actions tab
2. Click on the failed workflow run
3. Expand the failed job (build-base or build-runtime)
4. Review the error messages and build output

#### Manual Testing
Test locally before pushing:
```bash
# Test base image build
docker build -f Dockerfile.base -t test-base .

# Test runtime build with local base
docker build -f Dockerfile.runtime --build-arg BASE_IMAGE=test-base -t test-runtime .

# Test runtime container
docker run --rm test-runtime python --version
```

### Runtime Image Issues
```bash
# Force rebuild runtime image
docker build --no-cache -f Dockerfile.runtime -t ghcr.io/johannesbrandenburger/typst-mcp:latest .
```

### Documentation Problems
Check that the base image was built successfully:
```bash
docker run --rm ghcr.io/johannesbrandenburger/typst-mcp-base:latest ls -la /usr/local/share/typst-docs/
```

## CI/CD Best Practices

### Development Workflow
1. **Code changes**: Push normally - only runtime image builds
2. **Documentation changes**: Modify `Dockerfile.base` - both images build
3. **Testing**: Use PRs to test changes without pushing to registry
4. **Releases**: Create releases for production deployments

### Performance Optimization
- **Base image**: Builds rarely (~15 minutes), cached for runtime builds
- **Runtime image**: Builds quickly (~2 minutes) for code changes
- **Multi-platform**: Supports both AMD64 and ARM64 architectures
- **Caching**: Uses GitHub Actions cache for faster builds

### Monitoring and Maintenance
- **Watch build times**: Runtime builds should stay under 5 minutes
- **Check image sizes**: Base ~500MB, Runtime ~200MB
- **Regular updates**: Update base dependencies monthly
- **Security scans**: GitHub provides automatic vulnerability scanning

## File Structure
```
.
├── Dockerfile.base          # Base image with Typst docs
├── Dockerfile.runtime       # Runtime image using base
├── build.sh                 # Build helper script
├── .github/workflows/docker.yml  # CI/CD pipeline
├── DOCKER.md               # This documentation
└── README.md               # Main project README
```

## Quick Reference

### Common Commands
```bash
# Build everything
./build.sh

# Build only base image
./build.sh base

# Build only runtime (fast)
./build.sh runtime

# Push to registry
./build.sh push

# Run locally
./build.sh run
```

### CI/CD Commands
```bash
# Trigger base image rebuild
echo "# Trigger rebuild" >> Dockerfile.base
git add Dockerfile.base && git commit -m "trigger base rebuild"
git push

# Check workflow status
gh run list --repo BaptTF/typst-mcp

# Download workflow artifacts
gh run download <run-id> --repo BaptTF/typst-mcp
```