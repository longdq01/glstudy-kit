# GLStudy Frontend — Design System

> Source of truth for colors, typography, spacing, components, and patterns.
> All pages must follow these rules for visual consistency.

---

## 1. Color Tokens

Defined in `tailwind.config.ts`. **Always use the semantic class — never hardcode hex values.**

### Text

| Token class | Hex | Usage |
|---|---|---|
| `text-text-heading` | `#101828` | Page titles, card titles, bold labels, strong values |
| `text-text-body` | `#4a5565` | Subtitles, descriptions, secondary labels |
| `text-text-muted` | `#717182` | Timestamps, meta info, placeholder-like text |
| `text-primary` | `#155dfc` | Links, active states, brand accents |

### Backgrounds

| Token class | Hex | Usage |
|---|---|---|
| `bg-bg-main` | `#f9fafb` | Page background |
| `bg-input-bg` | `#f3f3f5` | Input field backgrounds |
| `bg-tab-bg` | `#ececf0` | Tab list backgrounds |
| `bg-primary` | `#155dfc` | Primary buttons, highlighted controls |
| `bg-btn-dark` | `#030213` | Dark action buttons (submit, play) |
| `bg-white` | `#ffffff` | Cards, panels, containers |

### Border

| Token class | Value | Usage |
|---|---|---|
| `border-black/10` | `rgba(0,0,0,0.1)` | All card/item borders |
| `border-black/5` | `rgba(0,0,0,0.05)` | Subtle inner borders (stat mini-cards) |

### Status / Level colors (use via Tailwind utilities, not tokens)

| Purpose | Background | Text |
|---|---|---|
| Beginner / Success | `bg-green-100` | `text-green-700` |
| Intermediate / Warning | `bg-yellow-100` | `text-yellow-700` |
| Advanced / Danger | `bg-red-100` | `text-red-700` |
| Info / Category | `bg-primary/10` | `text-primary` |
| Error messages | `bg-red-50 border-red-200` | `text-red-600` |
| Success messages | `bg-green-50 border-green-200` | `text-green-700` |

### Icon backgrounds (ad-hoc, not in tokens — consistent Tailwind utilities)

| Color | Background | Icon color |
|---|---|---|
| Blue | `bg-blue-50` | `text-blue-500` |
| Green | `bg-green-50` | `text-green-500` |
| Orange | `bg-orange-50` | `text-orange-500` |
| Purple | `bg-purple-50` | `text-purple-500` |
| Red | `bg-red-50` | `text-red-500` |

---

## 2. Typography

### Scale

| Role | Class | Weight | Size |
|---|---|---|---|
| Page heading | `text-2xl font-bold text-text-heading` | 700 | 24px |
| Section heading | `text-xl font-bold text-text-heading` | 700 | 20px |
| Card title | `text-base font-semibold text-text-heading` | 600 | 16px |
| Body strong | `text-sm font-semibold text-text-heading` | 600 | 14px |
| Body | `text-sm text-text-body` | 400 | 14px |
| Caption / meta | `text-xs text-text-muted` | 400 | 12px |
| Label (form) | `text-sm font-medium text-text-heading` | 500 | 14px |

### Font Family

`Inter` — loaded via `tailwind.config.ts` fontFamily: `{ sans: ['Inter', 'sans-serif'] }`

---

## 3. Border Radius

| Token | Size | Usage |
|---|---|---|
| `rounded-full` | 9999px | Badges, pills, avatars, toggle circles |
| `rounded-2xl` | 16px | Main cards (`Card` component), page-level containers |
| `rounded-xl` | 12px | Nested items within cards, secondary panels |
| `rounded-lg` | 8px | Buttons, small icon containers, inputs |
| `rounded-md` | 6px | Code chips, small tags (e.g. transcript timestamps) |

> **Rule**: All top-level white boxes (`Card` component) use `rounded-2xl`. Nested clickable items inside a Card use `rounded-xl`. Buttons use `rounded-lg`.

---

## 4. Spacing

### Page layout

```
Layout main: mx-auto max-w-7xl px-4 py-8 sm:px-6 lg:px-8
Page content wrapper: space-y-6
Two-column grid: grid grid-cols-1 gap-6 lg:grid-cols-3
```

### Card internals (via `Card` component)

```
CardHeader:  p-6
CardContent: p-6 pt-0
```

### Manual card items (nested inside Card)

```
p-4   — standard nested item
p-3   — compact nested item (e.g. test cards, related videos)
```

### Common gap values

| Context | Gap |
|---|---|
| Stat cards grid | `gap-4` |
| Course cards grid | `gap-3` |
| Lesson grid (shadowing list) | `gap-5` |
| Items within a card | `space-y-3` |
| Form fields | `space-y-4`, label+input: `space-y-1.5` |
| Icon + text inline | `gap-2` or `gap-3` |

---

## 5. Components

### `Card`

```tsx
import { Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter } from '@/components/ui/card'
```

- Base: `rounded-2xl border border-black/10 bg-white shadow-sm`
- `CardTitle` renders `text-base font-semibold text-text-heading`
- `CardDescription` renders `text-sm text-text-muted`

### `Badge`

```tsx
import { Badge } from '@/components/ui/badge'
```

Base: `rounded-full px-2.5 py-0.5 text-xs font-semibold`

| Variant | Use case |
|---|---|
| `default` | Category tags (Shadowing, Từ vựng...) — blue |
| `beginner` | Beginner level — green |
| `intermediate` | Intermediate level — yellow |
| `advanced` | Advanced level — red |
| `premium` | Premium membership — amber gradient |
| `outline` | Neutral outlined pill |
| `secondary` | Gray background pill |

### `Button`

```tsx
import { Button } from '@/components/ui/button'
```

| Variant | Use case |
|---|---|
| `default` (dark) | Primary CTA, submit, dark action buttons |
| `outline` | Secondary actions, social login |
| `ghost` | Navigation, icon-only buttons |
| `link` | In-line text links |

| Size | Height | Use case |
|---|---|---|
| `sm` | 32px | Compact actions inside cards |
| `default` | 40px | Standard buttons |
| `lg` | 48px | Form submit, hero CTA |
| `icon` | 40×40px | Square icon buttons |

> **Rule**: Always use `<Button>` — never write manual `<button className="...">` for interactive controls.

### `Progress`

```tsx
import { Progress } from '@/components/ui/progress'
```

Default: `h-2 rounded-full bg-gray-200` with `bg-primary` indicator.
Override height via `className`: `h-1.5` (thin), `h-2` (standard), `h-2.5` (thick).

### `Avatar`

```tsx
import { Avatar, AvatarImage, AvatarFallback } from '@/components/ui/avatar'
```

Always provide `AvatarFallback` with initials as fallback for broken/missing images.

### `Tabs`

```tsx
import { Tabs, TabsList, TabsTrigger, TabsContent } from '@/components/ui/tabs'
```

Two tab styles in use:

1. **Underline style** (profile page): default shadcn behavior
2. **Pill style** (shadowing lesson page):
   ```tsx
   <TabsList className="w-full rounded-xl bg-gray-100 p-1">
     <TabsTrigger className="flex-1 rounded-lg data-[state=active]:bg-white data-[state=active]:shadow-sm" ...>
   ```

---

## 6. Layout Patterns

### Page header (back + title)

```tsx
<div className="flex items-center gap-4">
  <Button variant="ghost" size="icon" asChild>
    <Link href="/..."><ArrowLeft className="h-5 w-5" /></Link>
  </Button>
  <div>
    <h1 className="text-2xl font-bold text-text-heading">Page Title</h1>
    <p className="text-sm text-text-muted">Subtitle</p>
  </div>
</div>
```

### Two-column layout (main + sidebar)

```tsx
<div className="grid grid-cols-1 gap-6 lg:grid-cols-3">
  <div className="lg:col-span-2 space-y-4">...</div>
  <div>...</div>
</div>
```

### Sub-header bar (course context bar)

Used on `/shadowing/[id]` to show course progress below the navbar:

```tsx
<div className="-mx-4 -mt-8 sm:-mx-6 lg:-mx-8 sticky top-16 z-40 border-b border-black/10 bg-white">
  <div className="mx-auto flex h-14 max-w-7xl items-center justify-between px-4 sm:px-6 lg:px-8">
    ...
  </div>
</div>
```

### Nested item (inside Card)

```tsx
<div className="rounded-xl border border-black/10 p-4 hover:bg-gray-50 transition-colors">
  ...
</div>
```

For clickable items, wrap in `<Link>` instead of `<div>`.

---

## 7. Icon Usage

Icon library: **Lucide React** (`lucide-react`)

| Size | Class | Context |
|---|---|---|
| 16px | `h-4 w-4` | Inline with text, nav links |
| 20px | `h-5 w-5` | Standalone icons, icon buttons |
| 24px | `h-6 w-6` | Card decorative icons, large actions |

Icon containers:

```tsx
// Small (32px) — sidebar stats
<div className="flex h-8 w-8 items-center justify-center rounded-lg bg-blue-50">
  <BookOpen className="h-4 w-4 text-blue-500" />
</div>

// Medium (40px) — card stat icons, course icons
<div className="flex h-10 w-10 items-center justify-center rounded-xl bg-blue-500">
  <Video className="h-5 w-5 text-white" />
</div>
```

---

## 8. Interaction Patterns

### Hover states

```
Cards/links: hover:bg-gray-50 transition-colors
Buttons: handled by Button component variants
Icon buttons: hover:bg-gray-100 transition-colors
```

### Focus states

Handled by Button component (`focus-visible:ring-2 focus-visible:ring-primary`).
For custom interactive elements, add `focus:outline-none focus-visible:ring-2 focus-visible:ring-primary`.

### Active/selected states

```
Nav links: bg-primary/10 text-primary
Filter buttons (shadowing): bg-btn-dark text-white
Speed buttons: bg-btn-dark text-white
```

---

## 9. Forms

```
Input base: rounded-lg border border-black/10 bg-input-bg
Label:      text-sm font-medium text-text-heading
Error:      text-xs text-red-500 (below field)
Alert box:  rounded-lg bg-red-50 px-4 py-3 border border-red-200 text-sm text-red-600
Success:    rounded-lg bg-green-50 px-4 py-3 border border-green-200 text-sm text-green-700
```

---

## 10. Pages Reference

| Page | Route | Key components |
|---|---|---|
| Landing | `/` | Hero, Features, Stats, CTA |
| Login / Register | `/login` | Centered card layout, two-panel, Tabs |
| Dashboard | `/dashboard` | Stat cards, RecentLessons, CoursesGrid, Tests, Achievements, DailyGoals |
| Profile | `/profile` | AvatarCard, PersonalInfo, History, Achievements, Settings (Tabs) |
| Shadowing List | `/shadowing` | LessonGrid, ProgressBar, level Badge |
| Shadowing Lesson | `/shadowing/[id]` | Sub-header, VideoPlayer, Transcript/Vocab Tabs, RelatedVideos |

---

## 11. Anti-patterns (Do NOT do these)

```tsx
// ❌ Hardcoded hex colors
<p className="text-[#0a0a0a]">Title</p>

// ✅ Use semantic token
<p className="text-text-heading">Title</p>

// ❌ Manual badge styling
<span className="rounded-full bg-green-100 px-2 py-0.5 text-xs text-green-700">Beginner</span>

// ✅ Use Badge component
<Badge variant="beginner">Beginner</Badge>

// ❌ Manual button/interactive element
<button className="flex h-9 w-9 items-center justify-center rounded-lg border ...">

// ✅ Use Button component
<Button variant="ghost" size="icon"><Bell className="h-5 w-5" /></Button>

// ❌ Different border radius on same-level cards
<div className="rounded-xl border ...">   // some cards
<div className="rounded-2xl border ...">  // other cards

// ✅ Use Card component or pick one:
// Top-level containers → rounded-2xl
// Nested items → rounded-xl
```
