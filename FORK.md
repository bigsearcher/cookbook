# Fork notice — `mkisel-patches`

This is a personal fork of [`nextcloud/cookbook`](https://github.com/nextcloud/cookbook)
maintained at <https://github.com/bigsearcher/cookbook>. The default
branch is upstream `master`; the patches live on the
[`mkisel-patches`](https://github.com/bigsearcher/cookbook/tree/mkisel-patches)
branch, rebased on top of upstream tags (currently `v0.11.9`).

## Why this fork exists

Stock Cookbook freezes the browser indefinitely on the
all-recipes view (`#/`) when the cookbook contains tens of thousands of
recipes. My instance has ~30 750 imported recipes; the page never
finishes rendering, even on a desktop with 32 GB RAM. The freeze starts
*before* any DOM is painted — pure JavaScript work in the Vue layer.

Three independent performance issues compound. Each is fixed on this
branch.

## Upstream status

The performance work is proposed upstream as
[#3139](https://github.com/nextcloud/cookbook/pull/3139) and the backend
image fix as [#3140](https://github.com/nextcloud/cookbook/pull/3140).
Both are still open as of 2026-07-27, so this branch remains necessary.

## Rebase history

- `v0.11.6` — original branch point (2026-05-11)
- `v0.11.9` — current base (2026-07-27). Upstream migrated Vuex → Pinia
  and Webpack → Vite in between. The only conflict worth noting was the
  store call in `RecipeList.vue`: `store.dispatch('clearRecipeFilters')`
  became `legacyStore.clearRecipeFilters()`. Upstream independently
  applied the same `aria-label` binding fix carried here.
  The pre-rebase state is preserved as branch `mkisel-patches-0.11.6`.

Because upstream now builds with Vite, a deployment must ship **both**
`js/` and `css/` — stylesheets are emitted as separate hashed chunks
(including `vue-recycle-scroller`'s own CSS). Shipping only `js/` leaves
the chunk hashes inconsistent.

## What the patches change

The frontend changes are confined to three Vue components in `src/`.
`appinfo/routes.php`, the REST API at `/apps/cookbook/api/v1/*` and the
JSON shape of every response are **untouched** — mobile clients (iOS /
Android Cookbook apps) and any third-party API consumers see an
identical server. One PHP change exists (see "Backend" below); it only
rewrites a field inside the stored `recipe.json`.

### 1. Recipes are no longer made deeply reactive

`src/components/AppIndex.vue`, `src/components/SearchResults.vue`

```diff
- import { ref } from 'vue';
- const recipes = ref([]);
- recipes.value = response.data;
+ import { shallowRef, markRaw } from 'vue';
+ const recipes = shallowRef([]);
+ recipes.value = markRaw(response.data);
```

`ref()` wraps every nested object in a reactive `Proxy`. For 30k recipe
objects that is 30k `Proxy` allocations and full property-tracking setup
— several seconds of pure JS work on a fast desktop, longer on weaker
hardware. The list view never mutates individual recipe fields, only
replaces the whole array, so `shallowRef` + `markRaw` is sufficient and
costs effectively nothing.

### 2. Quadratic membership check replaced with `Set` lookup

`src/components/List/RecipeList.vue` — the `recipeObjects` computed used
to call, for each recipe:

```js
filteredRecipes.value
    .map((r) => r.recipe_id)
    .includes(rec.recipe_id)
```

That is `O(N²)` — for `N = 30 750` that is roughly one billion
operations, every time the computed re-runs. Replaced with a single
`Set` built once per run; lookup is `O(1)`:

```js
const filteredIds = new Set(filteredRecipes.value.map((r) => r.recipe_id));
return sortedRecipes.value.filter((r) => filteredIds.has(r.recipe_id));
```

This single change cut the freeze from ~30 seconds to imperceptible.

### 3. Sort no longer deep-clones the input

`src/components/List/RecipeList.vue`

```diff
- const rec = JSON.parse(JSON.stringify(recipes));
+ const rec = recipes.slice();
```

`Array.prototype.sort` mutates in place, so the original needs to be
copied; but `sort` only re-orders the *array*, it does not mutate
recipes themselves. Deep-cloning 30k objects via JSON round-trip
allocates millions of strings and objects for nothing. A shallow copy
is sufficient and orders of magnitude faster.

### 4. Recipe list is virtualized

`src/components/List/RecipeList.vue` — the old template rendered one
`<RecipeCard>` per recipe inside `<ul><li v-for>`, producing 30k DOM
nodes. Replaced with `<RecycleScroller>` from
[`vue-virtual-scroller`](https://github.com/Akryum/vue-virtual-scroller)
in `page-mode` + `grid-items` mode:

```vue
<RecycleScroller
    page-mode
    :items="visibleRecipes"
    :item-size="130"
    :grid-items="gridItems"
    :item-secondary-size="332"
    key-field="recipe_id"
>
    <template #default="{ item }">
        <RecipeCard :recipe="item" />
    </template>
</RecycleScroller>
```

Only the visible rows live in the DOM at any time. `gridItems` is
recomputed on `window` resize as
`floor((window.innerWidth − 300) / 332)`, where `300` is the rough
width of the Nextcloud navigation pane and `332` is one card cell
(300 px card + `2 × 1rem` margin).

`vue-virtual-scroller@^1.1.2` is added as a new runtime dependency in
`package.json` (Vue 2 line; the `2.x` releases of the library are for
Vue 3).

## Building and deploying

Same as upstream:

```bash
npm install   # ~1700 transitive deps; pause Nextcloud desktop sync first
              # if the working copy lives inside a synced folder
npm run build # ~20 s; output in js/
```

Run `rm -rf js css` before the final build, otherwise stale hashed
chunks from an earlier build ship alongside the current ones.

Then copy `js/*` **and** `css/*` into
`/var/www/nextcloud/apps/cookbook/`, plus `lib/Service/RecipeService.php`
for the backend change, `chown www-data:www-data`, and **bump
`<version>` in `appinfo/info.xml`** followed by
`sudo -u www-data php occ upgrade`. Without the version bump Nextcloud
keeps the old `?v=…` cache-busting hash on `<script src=…>` and browsers
serve the previous bundle from disk cache.

Set the version *above* the released upstream one it is based on
(e.g. `0.11.9.1` on top of `0.11.9`). Otherwise the appstore entry looks
newer, and the next `occ app:update` silently replaces this build with
stock upstream — which is exactly how the fork was lost between
2026-05 and 2026-07.

## Backend

`lib/Service/RecipeService.php` — after `imageService->setImageData()`
writes the downloaded image, the external URL left in `$json['image']`
is replaced with the user-relative path to the local `full.jpg` and
`recipe.json` is re-saved. Without it the edit form shows the source URL
and any field mutation triggers a re-download. The frontend renders
`imageUrl` via the `cookbook.recipe.image` API route rather than this
field, so the change is about JSON consistency, not rendering.

## Trade-offs and known regressions

- The recipe grid loses its CSS `flex-wrap` "snap to viewport" feel; the
  number of columns is a fixed integer recomputed only on `resize`.
  Acceptable in practice.
- Card height is fixed at 130 px. If a future card variant grows
  vertically, items will overlap or clip — bump `:item-size`.
- Large libraries still require the full recipe list payload over the
  wire. Backend pagination would help further but is out of scope for
  these patches.

## Upstream

Tracking <https://github.com/nextcloud/cookbook>. Rebase
`mkisel-patches` on each upstream tag; conflicts have so far been
limited to `package.json` / `package-lock.json`.
