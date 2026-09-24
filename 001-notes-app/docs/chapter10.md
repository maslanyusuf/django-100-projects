# 001 Notes App — Step 10: Live search (one URL, two renders)

## Step 9 check, answered

A `<form>` submission serializes the form's inputs, including the hidden `csrfmiddlewaretoken` field that `{% csrf_token %}` renders. An `hx-post` on a standalone button has no inputs to serialize, so no token reaches the server and `CsrfViewMiddleware` rejects the request with a 403.

In the browser you'd see **nothing happen**. By default htmx does not swap 4xx/5xx responses, so the page stays as-is. The failure only shows up in the Network tab (403) and as an `htmx:responseError` event in the console. The `X-CSRFToken` header on `<body>` fixes it because Django accepts that header as an alternative to the form field.

## Concept

One URL serves two representations:

- Normal navigation (address bar, refresh, link click) gets the **full page**.
- An htmx request gets only the **fragment** that replaces part of the page.

The branch is `request.htmx`, from the middleware you added in Step 9. Keeping one URL means search results are linkable, bookmarkable and work without JS.

The catch is caching. A browser or proxy caching `GET /?q=foo` doesn't know that two different bodies exist behind the same URL. The **`Vary`** response header lists the request headers that affect the response body. `Vary: HX-Request` tells caches to key on that header, so a fragment is never served to a full-page request (or the reverse). Django's `vary_on_headers` decorator adds it.

**Reading the trigger.** `hx-trigger="input changed delay:300ms, search"` means:

- `input` fires on every keystroke event.
- `changed` skips the request if the value didn't change.
- `delay:300ms` debounces: the request goes out only after 300 ms of silence.
- `search` is the event a `type="search"` input fires when the user clears it with the built-in ×.

**Swap semantics.** With `hx-target="#note-list"` and the default swap (`innerHTML`), htmx replaces the _contents_ of `#note-list`. That's why `_note_list.html` (cards and pagination) is the fragment and `#note-list` itself stays in the full-page template.

**`hx-push-url="true"`** pushes the request URL (`/?q=foo`) into browser history, so the URL reflects state and Back works.

## Build this

1. In `note_list`, choose the template by `request.htmx`: `notes/partials/_note_list.html` for htmx, `notes/note_list.html` otherwise. Same context for both.
2. Decorate the view with `@vary_on_headers("HX-Request")` (from `django.views.decorators.vary`).
3. On the search input, add `hx-get` (pointing at the list URL via `{% url %}`), the `hx-trigger` above, `hx-target="#note-list"` and `hx-push-url="true"`.
4. The input needs `name="q"` and its `value` prefilled from the `q` context variable, so a refreshed URL shows the query in the box.
5. Make sure `archived` survives a search. An `hx-get` on an input sends only that input's own value, so look up `hx-include` and include the hidden `archived` field.
6. Leave the pagination links as normal full-page links. The spec doesn't add htmx to them.
7. Verify:
   - Typing updates the list with no full reload; the Network tab shows the response has no `<html>`.
   - The URL updates to `/?q=...`. Refresh gives the same results with the input prefilled.
   - Back and Forward step through previous searches and show the right results. **If Back ever renders a bare fragment with no page chrome, stop and investigate.** htmx sends a special extra request header when it restores history after a cache miss, and `django-htmx` exposes it on `request.htmx`. Find it in the docs and decide what your view should do.
   - Clearing the box with × resets the list.
   - In `?archived=1` mode, searching keeps you in the archived set.
   - `curl -si -H "HX-Request: true" localhost:8000/` returns a fragment with `Vary` including `HX-Request`; without the header you get the full page.
   - With JS disabled, pressing Enter in the search form still works (plain GET).
8. Commit: `feat(001): live search with htmx`.

## Paste me

- The `note_list` view and its decorators.
- The search form/input markup from `note_list.html`.
- What Back did, and anything odd from the checks above.

## Check question

Why does `Vary: HX-Request` matter here, and what concretely breaks without it?
