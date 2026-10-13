# Fork notice — `mkisel-patches`

This is a personal fork of [`nextcloud/cookbook`](https://github.com/nextcloud/cookbook)
maintained at <https://github.com/bigsearcher/cookbook>. The default
branch is upstream `master`. The patches live on one branch per upstream
base, so a port never rewrites published history: the current one is
[`mkisel-patches-v0.11.11`](https://github.com/bigsearcher/cookbook/tree/mkisel-patches-v0.11.11).
`mkisel-patches` (and its copy `mkisel-patches-0.11.9`) is the v0.11.9
base, `mkisel-patches-0.11.6` the original one.

## Why this fork exists

Stock Cookbook froze the browser on the all-recipes view (`#/`) when the
cookbook holds tens of thousands of recipes (this instance: ~30 750).
Four problems compounded; upstream has since fixed three of them, the
fourth is still carried here.

| Problem | Status |
|---|---|
| Every recipe wrapped in a reactive `Proxy` (`ref`) | fixed upstream (`shallowRef` + `markRaw`, 0.11.11) |
| `O(N²)` membership check in `recipeObjects` | fixed upstream (`Set` lookup, 0.11.11) |
| Sort deep-cloned the list via JSON | fixed upstream (`slice()`, 0.11.11) |
| One DOM node per recipe (30k `<li v-show>`) | **this fork** — virtualized list |

## What the patches change

Frontend, `src/components/List/RecipeList.vue`: the `<ul><li v-for v-show>`
list is replaced by `<RecycleScroller>` from
[`vue-virtual-scroller`](https://github.com/Akryum/vue-virtual-scroller)
**3.x** (Vue 3) in `page-mode` + grid mode, fed by `visibleRecipes`
(the sorted recipes the filters let through). Only the visible rows are
in the DOM. The grid is measured, not assumed: the cell is the first
rendered card plus its margins, the column count is the list's own width
(`ResizeObserver`) over the cell width. `EmptyList` stays gated on the full list, so a
filter that matches nothing keeps the filter controls on screen.

Also: four `aria-label="t(...)"` attributes are bound (`:aria-label`) —
the literal string was shipped to the DOM instead of the translation.

Backend, `lib/Service/RecipeService.php`: after `setImageData()` stores a
downloaded image, the external URL left in `recipe.json#image` is
replaced with the user-relative path of the local `full.jpg`. Without it
the edit form shows the source URL and a field mutation re-downloads.

Routes, the REST API at `/apps/cookbook/api/v1/*` and the JSON shape of
every response are untouched — mobile clients see an identical server.

## Upstream status

Virtualization: [#3139](https://github.com/nextcloud/cookbook/pull/3139)
(written for Vue 2; upstream took its other three fixes). Backend:
[#3140](https://github.com/nextcloud/cookbook/pull/3140). Both open.

## Rebase history

- `v0.11.6` — original branch point (2026-05-11)
- `v0.11.9` — 2026-07-27 (Vuex → Pinia, Webpack → Vite). Branches
  `mkisel-patches` / `mkisel-patches-0.11.9`.
- `v0.11.11` — 2026-10-13. Upstream moved to **Vue 3** (`@nextcloud/vue` 9,
  Pinia 4, vue-router 4) in 0.11.10, so the virtualization was re-done
  for `vue-virtual-scroller` 3.x rather than rebased.

## Building and deploying

```bash
npm ci && npm run build   # outside any synced folder; output in js/ and css/
```

Run `rm -rf js css` before the final build. Ship `js/*` **and** `css/*`
to `/var/www/nextcloud/apps/cookbook/`, plus `lib/Service/RecipeService.php`
and `appinfo/info.xml`, `chown -R www-data:www-data`, then
`sudo -u www-data php occ upgrade`.

The version is set **above** the upstream release it is based on
(`0.11.11.x` on `0.11.11`). Otherwise the appstore entry looks newer and
the next `occ app:update` silently replaces this build with stock
upstream — which is how the fork was lost twice (2026-05, 2026-10).

## Trade-offs

- All cards must share one size (the scroller uses one cell size); today's
  RecipeCard has a fixed height.
- The full recipe list still comes over the wire in one response.
