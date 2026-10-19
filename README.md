# Astro check watch mode: project path filtering

Minimal reproduction for `astro check --watch` exiting immediately when the project directory name contains `node_modules`.

- `astro` 7.4.0
- `@astrojs/check` 0.9.10
- `pnpm` 12.14.0

## Reproduce

```sh
cd folder-name-contains-node_modules
pnpm install
pnpm astro check --watch   # exits right after "Watching for changes..."

cd ../folder-name-normal
pnpm install
pnpm astro check --watch   # keeps running
```

The two directories are identical except for the `name` in `package.json`.

## Actual behavior

The initial check runs and prints:

```text
Watching for changes...
```

The process then exits and returns to the shell prompt.

## Expected behavior

The process should remain running and recheck files when they change. An ordinary project directory should not be excluded merely because its name contains `node_modules`.

## Cause

The watcher in `@astrojs/check` tests the whole absolute path with `pathStr.includes('node_modules') || pathStr.includes('.git')` (`dist/index.js`), so the project root itself is ignored when its path contains either string. A directory name such as `folder-name-contains.git` is affected in the same way.
