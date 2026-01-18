# Papermark Self-Hosted Patches

This directory contains patches that are applied to the upstream Papermark source code during the Docker build process.

## Purpose

The patches in this directory are used to customize the upstream Papermark application for self-hosted deployments. These modifications are applied automatically during the Docker build process to ensure compatibility with custom domain configurations.

## How It Works

1. During the CI workflow (see `.github/workflows/build-and-push.yml`), the upstream Papermark source is cloned
2. Files from this `patches/` directory are copied over the corresponding files in the Papermark source, preserving directory structure
3. README files are automatically excluded to preserve Papermark documentation
4. The modified source is then built into the Docker image

## Current Patches

### middleware.ts

**Purpose**: Fix custom domain handling for `sors.no` domain

**Problem**: The upstream Papermark middleware has hardcoded domain checks that treat any non-Papermark domain as a "custom domain" for document sharing. Additionally, the upstream code uses insecure `includes()` checks that can match malicious domains like `evilpapermark.io` or `papermark.io.evil.com`. This causes both functional issues (redirects) and security concerns.

**Solution**: Modified the `isCustomDomain()` function to:
1. Include `sors.no` in the list of allowed primary domains
2. Use secure domain matching for all domains to prevent false positives

```typescript
function isCustomDomain(host: string) {
  return (
    (process.env.NODE_ENV === "development" &&
      (host?.includes(".local") || host?.includes("papermark.dev"))) ||
    (process.env.NODE_ENV !== "development" &&
      !(
        host?.includes("localhost") ||
        host === "papermark.io" ||           // <-- Exact match (security fix)
        host?.endsWith(".papermark.io") ||   // <-- Subdomain match (security fix)
        host === "papermark.com" ||          // <-- Exact match (security fix)
        host?.endsWith(".papermark.com") ||  // <-- Subdomain match (security fix)
        host === "sors.no" ||                // <-- Exact match for self-hosted
        host?.endsWith(".sors.no") ||        // <-- Subdomain match for self-hosted
        host?.endsWith(".vercel.app")
      ))
  );
}
```

**Security Improvements**: 
- Fixed upstream security vulnerability: Changed `host?.includes("papermark.io")` to `host === "papermark.io" || host?.endsWith(".papermark.io")` to prevent matching malicious domains like `evilpapermark.io` or `papermark.io.evil.com`
- Same fix applied to `papermark.com` domain checks
- Applied secure matching pattern to new `sors.no` domain from the start

**Effect**: With this patch, `datarom.sors.no` will be treated as a primary application domain and will properly show the login page, dashboard, and other application routes instead of redirecting to `papermark.com`.

## Adding New Patches

To add a new patch:

1. Identify the file in the upstream Papermark repository that needs modification
2. Copy the modified file to this `patches/` directory, maintaining the same relative path structure from the Papermark root
   - For files in the root (like `middleware.ts`), place directly in `patches/`
   - For files in subdirectories (like `lib/middleware/app.ts`), create `patches/lib/middleware/app.ts`
3. Document the patch in this README with:
   - The file being patched
   - The reason for the patch
   - What the patch does
   - Any side effects or considerations

**Note**: README files (README*.md) are automatically excluded from being copied to avoid overwriting Papermark's documentation.

## Maintenance

When updating to a new version of Papermark:

1. Review the upstream changes to files that are patched
2. Update the patches if necessary to maintain compatibility
3. Test the patched build thoroughly before deploying

## Technical Details

The patches are applied in `.github/workflows/build-and-push.yml` during the CI workflow:

```bash
# Uses rsync if available for efficient copying
rsync -av --exclude='README*.md' patches/ papermark-src/

# Falls back to secure find with null-termination if rsync is not available
cd patches
find . -type f ! -name 'README*.md' -print0 | while IFS= read -r -d '' file; do
  dest="../papermark-src/${file#./}"
  mkdir -p "$(dirname "$dest")"
  cp -v "$file" "$dest"
done
cd ..
```

This approach ensures that:
- Patches are applied before the build process
- Directory structure is preserved for patches in subdirectories
- README files are automatically excluded
- Special characters in filenames are handled safely (null-terminated find)
- The process works reliably across different environments
