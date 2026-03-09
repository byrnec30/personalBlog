+++
title = "Sovling my bug in React's render cycle"
date = "2026-02-25T12:00:00-08:00"
draft = false
tags = ["react", "nextjs", "zustand", "hydration", "debugging"]
category = "general"
+++


I hit a bug in my Zustand/React app, and solving it helped me to see exactly how React’s rendering lifecycle works.

I created a very basic app to log observations of native plant and animal species, and I wanted to put in a filter box to sort my observations by species and location.


<video controls preload="metadata" width="100%">
  <source src="{{< staticurl "post1/screenRecordingBlogPost1.mp4" >}}" type="video/mp4">
  <source src="{{< staticurl "post1/screenRecordingBlogPost1.m4v" >}}" type="video/mp4">
  <source src="{{< staticurl "post1/screenRecordingBlogPost1.mov" >}}" type="video/quicktime">
  Your browser does not support the video tag.
</video>


[Download the screen recording]({{< staticurl "post1/screenRecordingBlogPost1.mp4" >}})


My simple React and Next.js app has a Zustand store to manage its state. My Zustand store has three state fields:

- byId, a dictionary object that maps each observation to its Id number; it’s a handy way to look up an individual observation.
- allIds, an array of only the Ids of each observation in the database. This is a code example of how I used field as a straightforward way to refresh the data delivered to my components, by mapping each Id to its record:


```jsx
const observations = allIds.map((id) => byId[id])
```


- And lastly, filters, which stores the current active filters; elsewhere in my store I’ve set these to be ‘location’ and ‘species’, like you can see in the filter boxes above.


```jsx
export const useSpeciesStore = create<ObservationsState>()(
  (set) => ({
    byId: {},
    allIds: [],
    filters: {},
    setAll: (obs: Observation[]) =>
      set(() => {
        const byId: Record<number, Observation> = {}
        const allIds: number[] = []
        obs.forEach((o) => {
          byId[o.id] = o
          allIds.push(o.id)
        })
        return { byId, allIds }
      }),
    setFilters: (filters) =>
      set((state) => ({
        filters: { ...state.filters, ...filters },
      })),
  }),
  )
```


I defined some selectors in my Zustand SpeciesStore file. This seemed like a nice clean way to filter my data and return the observations with the correct species or location.


```jsx
export const selectObservationById =
  (id: number) => (state: ObservationsState) => state.byId[id]

export const selectAllObservations = (state: ObservationsState) =>
  state.allIds.map((id) => state.byId[id])

export const selectFilteredObservations = (state: ObservationsState) => {
  const { species, location } = state.filters
  return state.allIds
    .map((id) => state.byId[id])
    .filter((o) => {
      if (species && o.species !== species) return false
      if (location && o.location !== location) return false
      return true
    })
}
```


Only one of my selectors, selectObservationById, didn’t cause any errors; sadly I have no real need for this selector, as it’s just using my store’s byId state field that I have already. It is kind of useful for reusability, as I can just import selectObservationById and it’s probably easier than calling the state field function each time, plus I can put in some type checking there with TypeScript to make sure Id is a number. It also matches the other selectors format, which is nice for readability (if only they’d worked).

Here’s where I hit my bug.

![Screenshot 2025-10-05 at 10.47.52.png]({{< staticurl "post1/errorMessageBlog1.png" >}})

selectAllObservations and selectFilteredObservations both use built-in functions (.map() and .filter()), and after some digging I realised that they were causing issues because of React’s rendering lifecycle. Here’s a quick little reminder of how it works.

In this Next.js SSR setup, React starts the render lifecycle with a server phase. It first renders the component tree on the server and produces the plain HTML. A fresh store instance is created during the server render, and my Zustand selectors are created inside that render.

As soon as the plain HTML is sent, the store vanishes; it doesn’t persist between requests.

Then we have the hydration stage on the client side: the browser receives that HTML, and  then attaches event listeners, state, and interactivity on the client side. When this hydration happens, React’s goal is to verify that the HTML it’s re-creating matches what the server sent. A new Zustand store instance is created in the browser, and the selector runs again during hydration.

The problem is that Map and Filter functions produce a new array instance on every call. During the hydration phase, React compares the “server snapshot” (what was rendered on the server) with the “client snapshot” (what Zustand’s useSyncExternalStore hook gives it on the client). The problem is that the selectors which use .map() or .filter() output a fresh array each time, which makes React see the client and server snapshots as different objects, even though the contents are the same.

[Understanding React's render phase (Stackademic)](https://blog.stackademic.com/understanding-reacts-render-phase-a-simplified-guide-3a84ea7aacba)


<img src="{{< staticurl "post1/diagramBlog1.jpg" >}}" alt="The objects don’t look the same any more" style="width: 66.67%; height: auto;">

The objects don’t look the same any more

That’s why we’re getting this error about an infinite loop.

How did I fix this? Really easy - I put all my selectors on the client side instead. I made a filter function in a util file, and I import it when I need it. I can use the .map() and .filter() functions here no problem, because all this filtering only renders on the client side, so we never get this mismatch between the client and server snapshots.


```tsx
function getFilteredIds(list: Observation[], filters: Filters): number[] {
  return applyFilters(list, filters).map((o) => o.id)
}
```

```tsx
export function applyFilters(
  list: Observation[],
  filters: Filters
): Observation[] {
  return list.filter((obs) => {
    if (filters.species && !obs.species.toLowerCase().includes(filters.species.toLowerCase())) {
      return false
    }
    if (filters.location && !obs.location?.toLowerCase().includes(filters.location.toLowerCase())) {
      return false
    }
    return true
  })
}
```


We don’t get this same error in the store state fields, because when the store is initially created on the server side, the state fields remain empty. These fields will only be populated when the setAll function is called, after the client side has already been hydrated.

`byId` = `{}`

`allIds` = `[]`

`filters` = `{}`

The problem with these two selectors is that they create non-stable references during the hydration stage, which React doesn’t allow. I got away with it with my first selector, because all it’s doing is reading from the state.

Another fix that I tried was to see if I could memoize the selectors. The idea here is that we’re trying to stop the selector producing a different array instance between the server and client renders. This memoize wrapper means that the selector will reuse the same array reference as long as nothing changes - and on the *first* render, nothing does change.

The first render is the only important one here, because when React hydrates, it only expects the initial client render to produce the exact same DOM structure as it got from the server. After hydration, it stops comparing the server completely. Then we can still use this selector to filter and produce different arrays, because after hydration everything is just a normal re-render.


```tsx
import { memoize } from 'proxy-memoize';

export const selectFilteredObservations = memoize((state: ObservationsState) => {
  const { species, location } = state.filters;
  const out: Observation[] = [];
  for (const id of state.allIds) {
    const o = state.byId[id];
    if (!o) continue;
    if (species && !o.species.toLowerCase().includes(species.toLowerCase())) continue;
    if (location && !o.location.toLowerCase().includes(location.toLowerCase())) continue;
    out.push(o);
  }
  return out;
})
```


Bug Fix 1, where we derive the view of the data on the client side (outside of the store), is probably the simplest model to understand: the store holds the state, and you use the UI to work out the view. This approach is hydration-safe by default if the derivation only runs on the client, after hydration. React never has to compare server and client snapshots of a derived array, because the store itself remains stable during hydration. The tradeoff might be that you’re doing extra work as filtering happens every render (not an issue in an app as small as this), and you could end up duplicating logic if you’re reusing the same filters elsewhere.

Bug Fix 2, where you keep the selectors in the store but make the stable using memoization, could give you:

- a single source of the selected data, that you can reuse everywhere
- better performance? you’re not recomputing each time
- cleaner components, if you’re keeping all the selector logic in the store

But the tradeoff for this fix is that you’re introducing that bit more complexity. It’s easier to create subtle hydration or subscription issues if a selector returns unstable references; memoization doesn’t guarantee server and client will produce the same reference unless the selector inputs are identical, and you need to be careful that you’re not triggering mismatches or extra renders.
