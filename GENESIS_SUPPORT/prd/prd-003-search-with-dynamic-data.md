# PRD-003: Search Correctness with Dynamic Data

## Problem / requirement

The current fuzzy search operates on a static dataset loaded once at page load. The name list (`Object.keys(data)`) is computed once and never changes. The search algorithm (Levenshtein distance with Polish-locale collation) assumes a fixed set of names.

When data becomes dynamic — names arriving at runtime via realtime updates — the search must incorporate newly arrived names into its searchable set. A user who is actively typing must be able to find a name that was added after they opened the page. The search must remain correct: Polish diacritics-insensitive, fuzzy, returning the top closest matches with their associated dates.

The existing search UX contract must be preserved: the user types a name, and the top fuzzy matches with their dates appear, updated per keystroke.

## Success criteria

- A name added via realtime propagation (PRD-001) must be findable by the fuzzy search within the same session, without page reload, within the same time window as the propagation (≤60 seconds of the data arriving at the client).
- Polish diacritics-insensitive matching must work for all names, including newly added ones (e.g., searching "Zofia" must match "Żofia" and vice versa).
- The search must continue to return the top closest matches by Levenshtein distance, with a distance threshold, updated per keystroke.
- Search must not degrade below perceptible responsiveness (results must appear within 200ms of a keystroke) at the current dataset scale (~5000 names) and at 2x that scale (~10000 names).

## Out of scope

- Changing the search algorithm (e.g., replacing Levenshtein with a different fuzzy matching approach).
- Server-side search (search remains client-side unless a separate decision determines otherwise).
- Search suggestions, autocomplete, or search-as-you-type beyond the current behaviour.
