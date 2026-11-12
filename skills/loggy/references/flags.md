# Feature flags with Loggy

## In the code

The second argument is the default, and it also sets the flag's type. A flag that doesn't exist, has another type, is archived or can't be loaded gives the default, so the app keeps working either way.

```tsx
// Client Component
import { useFlag } from "loggy-next";

const enabled = useFlag("new-checkout");                          // boolean, defaults to false
const label = useFlag("checkout-button-copy", "Buy now");         // string
const limits = useFlag("plan-limits", { maxProjects: 3 });        // json
```

```ts
// Server Component, Route Handler or Server Action
import { getFlag, getFlags } from "loggy-next";

const enabled = await getFlag("new-checkout", false, { key: user.id, plan: user.plan });
const all = await getFlags({ key: user.id, plan: user.plan }); // every flag, to pass to Client Components
```

A few details about reading flags:

- `useFlag` returns the default during server rendering and until flags load. When the first paint matters, read the flag on the server and pass it down.
- A page that calls `getFlag` renders per request. With Cache Components, wrap the component that reads it in `<Suspense>`.
- Changes reach busy servers within 5 seconds, and open pages within 30 seconds.

## Context: who a flag is for

- **What it is.** Rules target a context: a `key` that identifies someone, plus any attributes, such as `plan`, `email`, `country` or `company`. Values are text, numbers, booleans or lists of those.
- **On the server**, pass it to `getFlag` or `getFlags`.
- **In the browser**, give it to `<Loggy context={{ key: user.id, plan: user.plan }} />`, or call `identify({ ... })` from `loggy-next` when someone signs in or out.
- **Without a key**, the browser uses a random id it keeps, so anonymous visitors still get stable rollouts.
- **Before writing a rule** on an attribute, check in the code that the app actually passes that attribute.

## Kinds and variations

| Kind | Variations |
|---|---|
| `boolean` | Always `true` (index 0) and `false` (index 1). |
| `string` | Default `control`, `treatment`. |
| `number` | Default `0`, `1`. |
| `json` | Default `{}`, `{"enabled": true}`. |

- Rules point at variations by index.
- `set_flag_variations` keeps rules pointing at the right value when variations move. A variation can't be removed while any environment still serves it.
- A flag marked `server_only` is never sent to browsers, so `useFlag` always gets the default for it.

## Environments

- **Separate settings.** Each flag has its own settings in each environment.
- **What the app reports.** `VERCEL_ENV` on Vercel (`production`, `preview`, `development`), otherwise `NODE_ENV`, or the `environment` prop. An environment Loggy doesn't have gets production's settings.
- **Defaults.** New projects have production, preview and development.
- **Comments.** Production asks for a comment with every change. `list_environments` shows which environments do.

## Targeting JSON

`get_flag` ends with the environment's targeting as JSON and its version. `set_flag_targeting` takes the whole object back, with that version.

```json
{
  "on": true,
  "offVariation": 1,
  "rules": [
    {
      "id": "staff",
      "description": "Our team first",
      "conditions": [{ "attribute": "email", "operator": "endsWith", "values": ["@example.com"] }],
      "serve": { "variation": 0 }
    },
    {
      "id": "beta",
      "conditions": [{ "attribute": "key", "operator": "inSegment", "values": ["beta-testers"] }],
      "serve": { "variation": 0 }
    }
  ],
  "fallthrough": { "rollout": { "weights": [10000, 90000] } }
}
```

The fields:

- **`on`.** While off, everyone gets `offVariation`. While on, the rules are checked in order and the first match decides. Contexts that match no rule get `fallthrough`.
- **`conditions`.** All of them must match. `values` are always strings.

  | Operators | Compare |
  |---|---|
  | `in`, `notIn`, `contains`, `notContains`, `startsWith`, `endsWith` | Text |
  | `lt`, `lte`, `gt`, `gte` | Numbers |
  | `before`, `after` | Dates, in ISO 8601 |
  | `inSegment`, `notInSegment` | Membership; the values are segment keys |

- **`serve`.** Either `{ "variation": <index> }`, or `{ "rollout": { "weights": [...], "bucketBy": "<attribute>" } }`.
  - `weights` has one number per variation, in thousandths of a percent, and they add up to `100000`.
  - Bucketing uses `key` unless `bucketBy` names another attribute. Each person keeps their bucket, so raising 10% to 25% only adds people.
- **Rule ids.** Keep the ids of existing rules. A new rule needs a short id that no other rule in the flag uses.

### Recipes

- **Turn it on for everyone:** `set_flag_on` with `on: true`, if the default already serves the value you want. Otherwise set `fallthrough` to `{ "variation": 0 }`.
- **Gradual rollout:** set `fallthrough` to `{ "rollout": { "weights": [10000, 90000] } }`, then raise it later: 25000/75000, 50000/50000, and finally `{ "variation": 0 }`.
- **Only for some users:** add a rule that serves the new value, and leave `fallthrough` on the old one.
- **Kill switch:** `set_flag_on` with `on: false`. It keeps the rules for when the flag is turned back on.

## Segments

A segment is a reusable group that rules target with `inSegment`. A context is in the segment if one of these holds, unless its key is excluded:

- its key is in `included`;
- it matches every condition of any of the segment's rules.

To work with segments:

1. `create_segment` makes an empty segment.
2. `get_segment` shows its version.
3. `update_segment` takes `add_included` / `remove_included` (and the same for excluded) for single keys. `included` and `excluded` replace a whole list, and `rules` replaces the rules. Segment rules can't use other segments.

A segment that a flag still uses can't be deleted.

## Did a flag value bring errors?

`get_flag` with the environment shows, for the last 24 hours:

- the reads of each value;
- the errors per 1,000 reads, compared with the value served while off (the baseline);
- the problems that come with each value.

To see the logs themselves, search `flag:<key>=<value> level:error`.

## Guards

`set_flag_guard` watches a flag in one environment. It acts when a value's error rate reaches `max_ratio` times the baseline's, with at least `min_errors` errors, within `window_minutes` (30, 60 or 120). When it acts, it alerts the chosen channels, and with `turn_off` it also turns the flag off. It is a good companion to a rollout in production.
