# vinext app

This project was created with create-vinext-app.

## Scripts

- `pnpm run dev` starts the vinext dev server.
- `pnpm run build` builds the Cloudflare Worker output.
- `pnpm run start` previews the built Worker locally.
- `pnpm run deploy` deploys the Cloudflare Worker.

## Known dev server warnings

`pnpm run dev` prints the warnings below. They only show up in development and the app still works. Versions involved: vinext 1.4.6, lucide-react 1.77.0, @vitejs/plugin-rsc 0.5.39, @cloudflare/vite-plugin 2.0.0-rc.3, React 19.3.1.

### `[vinext:optimize-imports] skipping "lucide-react": could not resolve specifier "Monitor" in barrel export map`

vinext reads the CommonJS build of lucide-react as the barrel file, so it finds no ESM exports.

- To find a package's entry, `resolveExportsValue` in vinext's `optimize-imports` plugin checks `exports` conditions in the order `react-server → node → import → module → default`.
- lucide-react's `exports["."]` maps `"node"` to `./dist/cjs/lucide-react.js`, so vinext picks the CommonJS file ahead of the ESM one (`./dist/esm/lucide-react.mjs`).
- The CommonJS file has no `export` statements, so the export map comes out empty and every named import fails to resolve. `Monitor` is exported by the ESM barrel, so the icon name is not the problem.
- As a result, the barrel import is not rewritten to a direct import of the icon module, and the whole barrel gets loaded instead. This bug is in vinext and affects any package whose `node` condition points to CommonJS while `import`/`module` points to ESM (rxjs shows the same symptom in cloudflare/vinext#740).

### `(rsc) warning: client component dependency is inconsistently optimized`

lucide-react 1.x marks `dist/esm/Icon.mjs` and `dist/esm/context.mjs` with `"use client"`, and lucide-react is pre-bundled in the `rsc` environment.

- With `@cloudflare/vite-plugin`, the `rsc` environment runs in workerd and its dependencies are pre-bundled by `optimizeDeps`, so modules get ids like `Icon.mjs?v=<hash>`.
- `@vitejs/plugin-rsc` needs client references to have the same id in the server and client environments. When a `"use client"` module sits inside a pre-bundled dependency, the ids can differ, and plugin-rsc warns about it.
- Possible workaround: exclude the package from pre-bundling in the `rsc` environment only. Excluding it from the client environment too would load every icon module one by one in dev.

  ```ts
  export default defineConfig({
    environments: {
      rsc: { optimizeDeps: { exclude: ["lucide-react"] } },
    },
  });
  ```

- Fixing the previous warning may also make this one go away, because the rewritten import would point at the icon module directly instead of the pre-bundled barrel. This has not been verified.

### `eval() is not supported in this environment`

The SSR environment runs in workerd, and the Workers runtime does not allow `eval()` or `new Function()`.

- `childEnvironments: ["ssr"]` in `vite.config.ts` runs SSR inside workerd together with `rsc`.
- The development build of the React Flight client (`react-server-dom-webpack`) uses `eval()` to rebuild Server Component call stacks for debugging.
- React never uses `eval()` in production, so this cannot be fixed while developing on workerd and can be ignored. The only effect is less complete server-side stack traces in dev errors.
