# 001 Notes App — Step 11: Pin / archive / delete with HTMX

## Step 10 check, answered

`Vary: HX-Request` declares that the response body depends on that request header, so caches must key on it. Without it, a cache can store the **fragment** returned for an htmx `GET /?q=foo` and later serve it for a normal navigation to the same URL (Back, refresh, a pasted link). The user then sees a bare, unstyled fragment with no `<html>`. The reverse also happens: a cached full page gets injected into `#note-list`.

## Concept

For a mutation, the htmx pattern is: POST, server changes state, server returns the **HTML that should now exist** in place of the old element.

- Pin returns the re-rendered card. `hx-swap="outerHTML"` replaces the old `<article>` with it, so `_note_card.html` must have a **single root element**.
- Archive and delete return an **empty body**. `outerHTML` swapping an element with nothing removes it from the DOM.
- The empty response must be status **200**, not 204. htmx deliberately does not swap on 204 No Content, so the card would stay.
- `hx-target="closest article"` resolves relative to the element that fired the request, so each button acts on its own card.
- `hx-confirm` shows a native `confirm()` dialog and aborts the request if the user cancels.
- Non-htmx requests keep the redirect fallback from Step 7. The same URL, method and view serve both.

**`auto_now` and `update_fields`.** `save(update_fields=[...])` writes only the listed columns. `auto_now` recomputes `updated_at` in `pre_save`, but the value is only persisted if `"updated_at"` is in the list. Miss it and the pin changes but `updated_at` doesn't.

Toggling is a read-modify-write in Python, so two concurrent clicks can race. That's accepted for this project; `F()` expressions arrive in 006.

## Build this

1. **Pin view.** Toggle `is_pinned`, then `note.save(update_fields=["is_pinned", "updated_at"])`. If `request.htmx`, render `notes/partials/_note_card.html` with `note` in context. Otherwise redirect to `notes:list`.
2. **Archive view.** Toggle `is_archived` (save with the right `update_fields`). If `request.htmx`, return `HttpResponse("")` with status 200. Otherwise redirect.
3. **Delete view.** Delete the note. htmx: `HttpResponse("")`, status 200. Otherwise the flash message and redirect from Step 7.
4. In `_note_card.html`, add attributes to the **card's** Pin / Archive / Delete buttons per the spec table:

| Button  | Attributes                                                      |
| ------- | --------------------------------------------------------------- |
| Pin     | `hx-post`, `hx-target="closest article"`, `hx-swap="outerHTML"` |
| Archive | same as Pin                                                     |
| Delete  | same, plus `hx-confirm` with a short message                    |

Point `hx-post` at the right route with `{% url %}`. Keep the existing `<form method="post">` wrappers so the non-JS fallback still works. 5. Leave the buttons on `note_detail.html` as plain forms. There's no card to swap there, and the redirect fallback is correct on that page. 6. Pin returns just the card, so the list is not reordered until the next load. That's per spec, not a bug. 7. Experiment: temporarily return status 204 from archive, click, and watch the card fail to disappear while the Network tab shows a successful request. Then switch back to 200. 8. Verify:

- Pin swaps the card in place with no full reload. After a refresh it sorts first, and `updated_at` has changed (check in admin).
- Archive removes the card; it appears under `?archived=1`. Unarchiving there removes it from the archived list.
- Delete asks for confirmation. Cancel does nothing; OK removes the card, and a refresh confirms it's gone.
- With JS disabled, all three buttons still work via redirect.

9. Commit: `feat(001): htmx pin, archive and delete`.

## Paste me

- The three views (pin, archive, delete).
- `_note_card.html`.
- Anything that misbehaved, especially in the archived view or with JS disabled.

The HTMX phase ends with this step. The **HTMX-phase checkpoint** against the Testing and Security checklists opens the Step 12 file.

## Check question

What does htmx do with a 204 response, and how does an empty 200 plus `hx-swap="outerHTML"` end up removing the card?
