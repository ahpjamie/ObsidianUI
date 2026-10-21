# Obsidian UI

A Blade and Alpine component library for Laravel and Livewire apps: a near-black dark mode with a matching light mode, six accent colours, glass overlays, and mobile patterns (a floating dock, and bottom sheets for menus and modals).

Components are anonymous Blade components used as `<x-ob::…>`. Behaviour ships as one Alpine plugin. It has no npm dependencies of its own: it uses the Alpine that Livewire bundles.

## Requirements

- PHP 8.3+
- Laravel 13
- Livewire 4 (provides Alpine and its focus, anchor, collapse, intersect, persist and sort plugins)
- Tailwind CSS v4, built with Vite

## Installation

### 1. Require the package

Add the repository to your app's `composer.json`, then require it:

```json
"repositories": [
    { "type": "vcs", "url": "git@github.com:<org>/ObsidianUI.git" }
]
```

```bash
composer require ahp/obsidian
```

The service provider is auto-discovered. It registers the `ob` component prefix and a local-only showcase at `/_obsidian`.

### 2. Import the CSS

In `resources/css/app.css`, import Obsidian right after Tailwind:

```css
@import 'tailwindcss';
@import '../../vendor/ahp/obsidian/resources/css/obsidian.css';

@source '../views';
```

`obsidian.css` defines the design tokens as Tailwind theme values (`bg-canvas`, `bg-surface-1`, `text-fg-2`, `border-hairline`, `bg-accent`, …). It also adds the `glass`, `scrim` and `focus-ring` utilities, loads the bundled Inter and JetBrains Mono fonts, and tells Tailwind to scan the package's own views.

Remove any other UI kit or theme plugin (for example DaisyUI): its theme variables and component class names collide with Obsidian's.

### 3. Register the Alpine plugin

In `resources/js/app.js`:

```js
import obsidian from '../../vendor/ahp/obsidian/resources/js/obsidian.js';

document.addEventListener('alpine:init', () => window.Alpine.plugin(obsidian));
```

### 4. Apply the theme before first paint

In your layout's `<head>`, before `@vite`, add the theme script. Pass the signed-in user's saved preferences if you store them (see [Theme preferences](#theme-preferences)), or nothing to use the defaults (`system`, `iris`):

```blade
<x-ob::theme-script :appearance="$user?->appearance" :accent="$user?->accent" />
@vite(['resources/css/app.css', 'resources/js/app.js'])
```

Both props take plain strings (use `->value` if you cast them to enums). The script sets `data-appearance`, `data-mode` and `data-accent` on `<html>` before the page paints, resolving `system` from the OS setting. The Alpine plugin then keeps them correct as the OS setting changes and across `wire:navigate` page swaps. Never put these attributes on a nested element: the tokens only resolve from `<html>`.

### 5. Use the app shell

A minimal authenticated layout:

```blade
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover" />
    <x-ob::theme-script />
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body>
    <x-ob::app-shell>
        <x-slot:sidebar>
            <x-ob::sidebar>
                <x-slot:brand>My app</x-slot:brand>
                <x-ob::sidebar.item href="{{ route('reports.index') }}" icon="inbox-stack" label="Reports" />
                <x-ob::sidebar.item href="{{ route('tasks.index') }}" icon="check-circle" label="Tasks" />
            </x-ob::sidebar>
        </x-slot:sidebar>

        <x-slot:dock>
            <x-ob::dock>
                <x-ob::dock.item href="{{ route('reports.index') }}" icon="inbox-stack" label="Reports" />
                <x-ob::dock.action icon="plus" label="New task" x-on:click="$dispatch('command-palette-open')" />
                <x-ob::dock.item href="{{ route('tasks.index') }}" icon="check-circle" label="Tasks" />
            </x-ob::dock>
        </x-slot:dock>

        {{ $slot }}
    </x-ob::app-shell>

    <x-ob::toasts />
</body>
</html>
```

The sidebar shows from 1024px up and the dock below it. Don't put `transform`, `filter` or `backdrop-filter` on the shell's `main` or its wrappers: modals and sheets are fixed-position and would anchor to that element instead of the viewport.

## Theme preferences

Obsidian doesn't store preferences. Your app decides where they live, for example two columns on `users`:

- `appearance`: `system` | `light` | `dark`
- `accent`: `iris` | `blue` | `teal` | `green` | `amber` | `rose`

Pass them to `<x-ob::theme-script>`. To change the theme live from a settings page, dispatch `ob-theme-apply` after saving:

```php
$this->dispatch('ob-theme-apply', appearance: 'dark', accent: 'teal');
```

The page can also apply a change optimistically with `$store.obTheme.apply({ appearance, accent })`. Partial updates keep the other value. The server value wins on the next full page load, and choices are also cached in `localStorage` under `ob-theme`.

## Events

| Event | Detail | Effect |
|---|---|---|
| `toast` | `{ message, variant?, action?: { label, href } }` | Shows a toast. From Livewire: `$this->dispatch('toast', message: 'Saved', variant: 'success')` |
| `modal-open` / `modal-close` | `{ name }` | Opens or closes `<x-ob::modal name="…">` or `<x-ob::sheet name="…">` |
| `command-palette-open` | — | Opens `<x-ob::command-palette>` (⌘K, Ctrl+K and `/` also open it) |
| `ob-theme-apply` | `{ appearance?, accent? }` | Applies a theme change |
| `ob-theme-cycle` | — | Cycles appearance dark → light → system for the current page |

Toast variants are `success`, `danger`, `warning`, `info` and `neutral`.

## Components

| Group | Components |
|---|---|
| Primitives | `icon`, `button`, `badge`, `avatar`, `avatar.stack`, `kbd`, `spinner`, `separator`, `tooltip` |
| Forms | `field`, `input`, `password`, `textarea`, `select`, `checkbox`, `switch`, `radio-group`, `segmented`, `combobox` |
| Layout | `app-shell`, `sidebar` (+ `.item`, `.group`), `dock` (+ `.item`, `.action`), `page-header`, `card`, `section`, `empty-state`, `stat` |
| Data | `table` (+ `.head`, `.row`, `.cell`), `property`, `inline-edit`, `kanban` (+ `.column`, `.card`) |
| Overlays | `dropdown` + `menu.item` / `menu.separator` / `menu.group`, `modal`, `sheet`, `toasts`, `command-palette` (+ `.group`, `.item`) |

Each component's props are declared in its `@props` block in `resources/views/components/`. Visit `/_obsidian` in your local environment to see every component in both modes and all accents. The showcase loads your app's `resources/css/app.css` and `resources/js/app.js` entries.

Conventions shared by all components:

- `wire:model*` and `x-model` go to the real form control. Field controls show the `$errors` message for their `wire:model` name (or `name`) automatically.
- `variant` is `primary` | `secondary` | `ghost` | `danger` and `size` is `sm` | `md` | `lg`. Icon names are Heroicons names (`inbox-stack`).
- Icon-only buttons require a `label`, which becomes the `aria-label` and the tooltip.
- `wire:model.live` on a `modal` or `sheet` syncs the open state to the server immediately, so `updated…` hooks run when a user closes it. Use it when closing must clear sensitive state.
- `inline-edit` calls a Livewire method with the new value and sends `null` for a cleared field. Make the method's parameter nullable and validate it.

## Development

```bash
composer install && vendor/bin/pest   # Blade component tests (Testbench)
npm install && npm test               # JS logic tests (Vitest, jsdom)
```

Fonts are Inter and JetBrains Mono under the SIL Open Font License (`resources/fonts/OFL.txt`).
