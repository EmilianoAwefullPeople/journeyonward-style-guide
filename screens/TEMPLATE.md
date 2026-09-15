# <Screen Name>

**Route name:** `<RouteNameAsRegisteredInAppNavigator>`
**File:** `src/screens/<Path>.tsx`
**Type:** Tab screen | Stack screen | Modal | Transparent modal

## Purpose
1–3 sentences: what this screen is for, which user flow/persona (see PRD personas) it serves.

## Entry points & navigation
- How the user gets here (which screen/action navigates to it, via `navigation.navigate('X', {...})`)
- Params it receives (from `route.params`)
- Where it navigates to / how it exits (back, replace, modal dismiss, etc.)

## Layout anatomy
Ordered breakdown of the screen's visual structure top to bottom (header, body sections, footer/CTA), referencing actual component names used in the file.

## States
- Loading
- Empty
- Populated
- Error
- Draft (if applicable)
- Any other toggles/conditional UI branches found in the code (list each with the condition that triggers it)

## Data & context dependencies
- Context/hooks consumed (`useApp()`, `useTheme()`, `useProgress()`, etc.) and which fields
- Services/managers called (e.g. `WeatherService`, `PersistenceManager`, `ImportLatestService`)

## Interactions
- Taps, gestures, animations, haptics found in the code (name the Reanimated/Gesture Handler primitives if used)

## Theming notes
- Light/dark-specific logic, if any (colors from `useTheme()` vs static `COLORS.*`)

## Reference UI skeleton
A stripped JSX + `StyleSheet` skeleton of the screen's **whole render tree** — real component names, real nesting/hierarchy, real key style values (spacing, colors, typography, dimensions), and a branch for every state listed above. Business logic (data fetching, handlers, animation internals, service calls) is replaced with a short `// ...` comment pointing at the real `file:line` — enough is kept that someone could rebuild the visual layout from this alone, without the file open. For very large screens, skeleton the full top-level structure but collapse repeated/list-item sub-trees to one representative instance.

```tsx
// source: src/screens/<Path>.tsx (skeleton — business logic stripped, see cited lines for real implementation)
```

**Key technique(s):** call out anything in the skeleton that isn't self-explanatory (e.g. a non-obvious layout trick, a platform branch, a recycling-pool gotcha), each with its own `file:line`.

## Screenshot
_Not yet captured. TODO: capture light + dark, states: [list]._

## Open questions / follow-ups
Anything ambiguous from static code reading alone that a live screenshot or a designer decision would need to resolve.
