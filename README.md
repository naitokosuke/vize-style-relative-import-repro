# vize-style-relative-import-repro

Minimal reproduction for a production-build failure in [vize](https://github.com/ubugeeei-prod/vize)'s Vite plugin.

A `<style>` block that imports a stylesheet by a path relative to its `.vue` file fails `vite build`. Since 0.516.0 the plugin gives each style block a root-relative id (`/src/components/Card.vue.__vize_style_0.css?…&vize-root-relative`). Vite's CSS pipeline takes that id as a file path, so it resolves `../styles/colors.css` against `/src/components` at the file-system root, a directory that does not exist.

## Steps

```sh
pnpm install
pnpm build
```

[`src/components/Card.vue`](./src/components/Card.vue):

```vue
<template>
  <p class="card">Card</p>
</template>

<style>
@import "../styles/colors.css";
</style>
```

## Expected

The build succeeds and the CSS contains `.card{color:tomato}`. This is what `@vitejs/plugin-vue` 6.0.10 and `@vizejs/vite-plugin` 0.515.0 produce.

## Actual

```
vite v8.3.5 building client environment for production...
[vize] Pre-compiling 1 Vue files... (1 changed, 0 cached, 0 removed)
[vize] Pre-compilation complete: 1 recompiled, 0 restored from disk, 0 reused, 0 failed (16.01ms, native batch: 3.02ms)
transforming...
Unable to resolve `@import "../styles/colors.css"` from /src/components
✓ 10 modules transformed.
✗ Build failed in 102ms
error during build:
Build failed with 1 error:

[plugin vite:css] /src/components/Card.vue.__vize_style_0.css?vue=&type=style&index=0&lang=css&vize-root-relative=&vize-file=%2Fsrc%2Fcomponents%2FCard%2Evue:undefined:NaN
Error: [postcss] ENOENT: no such file or directory, open '../styles/colors.css'
```

## Notes

- 0.516.0 is the first version that fails; 0.515.0 builds. The change came in [#10079](https://github.com/ubugeeei-prod/vize/pull/10079) ("stabilize scoped style ids across checkouts").
- The failure does not depend on `scoped`, and it is the same with `css.transformer: "lightningcss"`: `[lightningcss] ENOENT: no such file or directory, open '../styles/colors.css'`.
- An import from the project root (`@import "/src/styles/colors.css"`) builds.
- [`createStyleImportUrl`](https://github.com/ubugeeei-prod/vize/blob/v0.569.0/npm/builder/vite/src/utils/index.ts#L31-L51) makes the id root-relative in production. [`load`](https://github.com/ubugeeei-prod/vize/blob/v0.569.0/npm/builder/vite/src/plugin/load.ts#L209-L230) maps it back to the real file for Vize's own use, but `vite:css` resolves `@import` against the id it was given.

## Environment

vize 0.569.0, Vite 8.3.5, Vue 3.5.44, Node v26.12.1, pnpm 12.9.0, macOS (arm64)
