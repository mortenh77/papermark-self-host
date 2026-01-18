# Papermark Self-Hosted Patches

This directory contains patch files that are applied to the upstream Papermark source code during the Docker build process.

## How It Works

1. During the CI workflow (see `.github/workflows/build-and-push.yml`), the upstream Papermark source is cloned
2. Patch files (`.patch`) from this directory are applied using the standard `patch` command
3. The modified source is then built into the Docker image

## Current Patches

### middleware.patch

**Purpose**: Add support for `sors.no` domain

**Change**: Adds `sors.no` to the list of allowed primary domains in the `isCustomDomain()` function.

```diff
+        host?.includes("sors.no") ||
```

**Effect**: With this patch, `sors.no` and any `*.sors.no` subdomain will be treated as a primary application domain and will properly show the login page, dashboard, and other application routes instead of redirecting to `papermark.com`.

## Adding New Patches

To add a new patch:

1. Identify the file in the upstream Papermark repository that needs modification
2. Create a unified diff patch file:
   ```bash
   diff -u original_file.ts modified_file.ts > patches/filename.patch
   ```
3. Document the patch in this README with:
   - The file being patched
   - The reason for the patch
   - What the patch does

## Maintenance

When updating to a new version of Papermark:

1. Review the upstream changes to files that are patched
2. Test if patches still apply cleanly with `patch --dry-run`
3. Update patches if necessary
4. Document any conflicts or required changes

## Technical Details

The patches are applied in `.github/workflows/build-and-push.yml` during the CI workflow:

```bash
# Apply patches
cd papermark-src
for patch in ../patches/*.patch; do
  if [ -f "$patch" ]; then
    echo "Applying $(basename "$patch")..."
    patch -p0 < "$patch"
  fi
done
```

This approach ensures that:
- Minimal changes to upstream code
- Easy to review what's being modified
- Standard patch format for portability
- Clear separation between upstream and customizations
