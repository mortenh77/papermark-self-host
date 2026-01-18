# Papermark Self-Hosted Patches

This directory contains patches that are applied to the upstream Papermark source code during the Docker build process.

## Purpose

The patches in this directory are used to customize the upstream Papermark application for self-hosted deployments. These modifications are applied automatically during the Docker build process to ensure compatibility with custom domain configurations.

## How It Works

1. During the Docker build (see `Dockerfile.papermark`), the upstream Papermark source is cloned
2. Files from this `patches/` directory are copied over the corresponding files in the Papermark source
3. The modified source is then built into the Docker image

## Current Patches

### middleware.ts

**Purpose**: Fix custom domain handling for `sors.no` domain

**Problem**: The upstream Papermark middleware has hardcoded domain checks that treat any non-Papermark domain as a "custom domain" for document sharing. This causes `datarom.sors.no` to be redirected to `papermark.com`.

**Solution**: Modified the `isCustomDomain()` function to include `sors.no` in the list of allowed primary domains:

```typescript
function isCustomDomain(host: string) {
  return (
    (process.env.NODE_ENV === "development" &&
      (host?.includes(".local") || host?.includes("papermark.dev"))) ||
    (process.env.NODE_ENV !== "development" &&
      !(
        host?.includes("localhost") ||
        host?.includes("papermark.io") ||
        host?.includes("papermark.com") ||
        host?.includes("sors.no") ||  // <-- Added for self-hosted deployment
        host?.endsWith(".vercel.app")
      ))
  );
}
```

**Effect**: With this patch, `datarom.sors.no` will be treated as a primary application domain and will properly show the login page, dashboard, and other application routes instead of redirecting to `papermark.com`.

## Adding New Patches

To add a new patch:

1. Identify the file in the upstream Papermark repository that needs modification
2. Copy the modified file to this `patches/` directory, maintaining the same relative path structure
3. Document the patch in this README with:
   - The file being patched
   - The reason for the patch
   - What the patch does
   - Any side effects or considerations

## Maintenance

When updating to a new version of Papermark:

1. Review the upstream changes to files that are patched
2. Update the patches if necessary to maintain compatibility
3. Test the patched build thoroughly before deploying

## Technical Details

The patches are applied in `Dockerfile.papermark` during the builder stage:

```dockerfile
# Apply self-hosting patches
COPY patches/ /tmp/patches/
RUN if [ -d "/tmp/patches" ]; then \
      cp -r /tmp/patches/* . 2>/dev/null || true; \
    fi
```

This approach ensures that:
- Patches are applied before the build process
- The build fails gracefully if patches directory doesn't exist
- Only tracked files in the patches/ directory are applied
