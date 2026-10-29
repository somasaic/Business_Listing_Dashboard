# Step 8 — Frontend Dashboard (React + Recharts)

> **What we build:** a single-page dashboard that fetches from FastAPI and shows
> stat cards, a **city-wise bar chart**, a **category-wise bar chart**, a
> **source-wise donut (pie) chart**, global filters, and a searchable,
> paginated listings table.
>
> **Why React + Recharts:** React is mandatory. Recharts is one of the three
> libraries the brief suggests. It's built *for* React (charts are components),
> it renders SVG (sharp at any size and styleable with CSS variables), and it's responsive.

Folder: `frontend/src/`

---

## 1. Component tree

```
App.jsx                         ← owns filter state; lays out the page
├── FilterBar                   ← City / Category / Source dropdowns + "Clear filters"
├── StatCards                   ← Total · Cities · Categories · Sources · % with phone
├── ChartCard "Businesses by city"        (Chart | Table toggle)
│   └── CountBarChart           ← vertical bars (horizontal on phones)
├── ChartCard "Businesses by category"
│   └── CountBarChart horizontal
├── ChartCard "Businesses by source"
│   └── SourcePieChart          ← donut + legend with counts and %
└── ListingsTable               ← search, pagination, raw rows

hooks/useDashboardData.js       ← fetches summary + 3 counts for the current filters
hooks/useMediaQuery.js          ← "is the screen narrow?"
api/client.js                   ← fetch wrapper, base URL from VITE_API_URL
```

**Why this split:**
* **Presentational components** (`StatCards`, `CountBarChart`, `SourcePieChart`) only receive props and render. They're easy to reuse and reason about.
* **Data fetching** lives in a **custom hook**, not inside components.
* **One reusable `CountBarChart`** serves both city and category charts, because the API returns the same `{label, count}` shape for every dimension.
* **One `ChartCard`** provides the title, total, empty state and table view for every chart.

---

## 2. Talking to the API (`api/client.js`)

```js
const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:8000'

async function get(path, params = {}) {
  const query = new URLSearchParams(Object.entries(params).filter(([, v]) => v))
  const response = await fetch(`${API_URL}${path}${query.size ? `?${query}` : ''}`)
  if (!response.ok) throw new Error((await response.json()).detail || `Request failed (${response.status})`)
  return response.json()
}

export const api = {
  summary:       (f) => get('/api/dashboard/summary', f),
  cityCount:     (f) => get('/api/dashboard/city-count', f),
  categoryCount: (f) => get('/api/dashboard/category-count', f),
  sourceCount:   (f) => get('/api/dashboard/source-count', f),
  listings:      (p) => get('/api/listings', p),
}
```

* `import.meta.env.VITE_API_URL`: Vite's way to read `.env`, so a deployed build can point at a different API.
* `URLSearchParams` builds `?city=Mumbai&limit=5` safely (URL-encoding) and drops empty filters.
* API errors surface as JavaScript `Error`s carrying FastAPI's `detail` message.

---

## 3. Loading data: the `useDashboardData` custom hook

```js
export function useDashboardData({ city, category, source }) {
  const key = `${city}|${category}|${source}`
  const [state, setState] = useState({ key: null, data: null, error: null })

  useEffect(() => {
    let cancelled = false
    Promise.all([
      api.summary({ city, category, source }),
      api.cityCount({ category, source }),        // city chart ignores the city filter …
      api.categoryCount({ city, source }),        // … each chart is filtered by the OTHER dims
      api.sourceCount({ city, category }),
    ]).then(([summary, cities, categories, sources]) => {
      if (!cancelled) setState({ key, data: { summary, cities, categories, sources }, error: null })
    })
    return () => { cancelled = true }           // cleanup: ignore stale responses
  }, [key, city, category, source])

  return { data: state.data, error: state.error, loading: state.key !== key }
}
```

| React concept | Explanation |
|---|---|
| `useState` | holds the latest response *and which filter selection it belongs to* |
| `useEffect` with dependencies | re-fetches whenever a filter changes |
| **Effect cleanup** (`cancelled = true`) | if the user changes filters quickly, an older slow response can't overwrite the newer one (prevents a **race condition**) |
| `Promise.all` | the 4 requests run **in parallel**, so the wait is as long as the slowest one, not the sum |
| **Derived state** (`loading = state.key !== key`) | computed during render instead of setting state inside the effect. This avoids extra renders, as recommended by React's lint rules. |
| Custom hook | components stay focused on rendering. The data logic is reusable and testable. |

**Cross-filtering design:** when you pick *Mumbai*, the **city chart still shows
all cities**, with Mumbai highlighted and the others muted, so you keep context.
The category and source charts and the stat cards narrow to Mumbai.

---

## 4. Charts (Recharts)

### Bar chart (`CountBarChart.jsx`)
```jsx
<ResponsiveContainer width="100%" height={height}>
  <BarChart data={items} layout={horizontal ? 'vertical' : 'horizontal'}>
    <CartesianGrid stroke="var(--grid)" />
    <XAxis dataKey="label" />  <YAxis allowDecimals={false} />
    <Tooltip content={<ChartTooltip total={total} />} />
    <Bar dataKey="count" radius={[4, 4, 0, 0]} onClick={(e) => onSelect(e.label)}>
      {items.map((item) => <Cell key={item.label} fill={barColor(item.label)} />)}
      <LabelList dataKey="count" position="top" />
    </Bar>
  </BarChart>
</ResponsiveContainer>
```

* `ResponsiveContainer`: the chart resizes with its card.
* **Horizontal bars for categories:** 13 category names would collide on an x-axis. Horizontal bars give each label room. On phones the city chart switches to horizontal too (`useMediaQuery('(max-width: 640px)')`).
* Bars are **sorted by count** (the API orders them), so ranking is readable at a glance.
* `LabelList` prints the exact value on each bar, and the custom tooltip shows count and share.
* **Click a bar to filter** the whole dashboard by that city or category. Click again to clear.
* Only the data end of each bar is rounded (4px). The baseline end stays square.

### Donut chart (`SourcePieChart.jsx`)
* A donut (pie with `innerRadius`) shows a **part-to-whole** split. The total is printed in the centre.
* There are only 2 sources, which is the right number of slices for a pie. With many slices a bar chart is better, which is why city and category use bars.
* The legend is a list of buttons: swatch, name, count and **%**. Clicking one filters by source.
* **Colour follows the entity, not its rank.** Each source's colour comes from its index in the *full* source list, so filtering never repaints "OpenStreetMap" in a different colour.

### Colour and accessibility
* The palette is a **validated categorical palette** (blue `#2a78d6`, orange `#eb6834`, …), chosen to stay distinguishable for colour-blind viewers.
* Single-series bar charts use **one colour**. Colour isn't needed to tell bars apart, because the axis labels do that.
* Every chart card has a **Chart | Table toggle**. The table shows exact counts and shares as text, which helps screen-reader users and anyone who wants exact numbers.
* Text always uses neutral ink colours, never the series colour, for readable contrast.

---

## 5. Styling: clean and minimal (`index.css`)

* **Design tokens as CSS custom properties** (`--surface`, `--text-primary`, `--series-1`, `--grid` …) are defined once and used everywhere, including inside SVG charts (`fill="var(--series-1)"`).
* **Automatic dark mode** via `@media (prefers-color-scheme: dark)`. The same tokens get dark values, with steps chosen for the dark surface rather than simply inverted.
* Card layout with a CSS Grid: 2 columns on desktop and 1 on mobile, with the stat cards wrapping at 5 → 3 → 2 columns.
* System font stack, generous whitespace, hairline borders, recessive gridlines.
* Wide tables scroll horizontally inside their card, so the page itself never scrolls sideways on a phone.
* Visible keyboard focus rings (`:focus-visible`).

---

## 6. UX states handled

| State | What the user sees |
|---|---|
| First load | "Loading dashboard…" |
| Refetch after a filter change | previous charts stay visible plus a small "Updating…" note, with no flicker |
| API / MySQL down | red banner: *"Could not load dashboard data … Is the FastAPI server running on port 8000?"* |
| Filter with no results | "No listings match the current filters." |
| Table search | **debounced** 300 ms, so the API isn't called on every keystroke |
| Changing filters or search | table resets to page 1 |

---

## 7. Run it

```bash
cd frontend
cp .env.example .env      # VITE_API_URL=http://localhost:8000
npm install
npm run dev               # http://localhost:5173
npm run build             # production bundle in dist/
npm run lint              # oxlint – passes with 0 warnings
```

➡️ Next: [Step 9 — testing](09-testing.md)
