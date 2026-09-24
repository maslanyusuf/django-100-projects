# 001 Notes App — Step 7: Function-based views

## Step 6 check, answered

Nothing errors, which is the danger. With no namespace, `{% url "detail" 1 %}` looks the name up across the whole URL space. If two apps both define `detail` and both accept the same args, Django picks one (typically the later-defined pattern), so a link can silently point at the wrong app's page.

With `app_name = "notes"`, you write `{% url "notes:detail" 1 %}`. The namespace scopes the lookup to that app's URLconf, so the other app's `detail` can't interfere.

## Concept

A **function-based view (FBV)** is a function `(request: HttpRequest, *args) -> HttpResponse`. Class-based views arrive in 003; here you write the control flow by hand so you know what CBVs abstract.

Idioms in this step:

- **`get_object_or_404(Model, pk=pk)`** turns "no row" into an `Http404`, which Django renders as a 404 response instead of a 500.
- **`Paginator` / `get_page()`**: `paginator.get_page(request.GET.get("page"))` never raises. Garbage or missing values return page 1, out-of-range values return the last page. It's the right call for user-supplied input.
- **`ModelForm`** builds a form from a model. `Meta.fields` must be an explicit list, never `"__all__"`, or a later model field silently becomes user-editable. `form.is_valid()` runs validation, `form.save()` writes the instance.
- **Post/Redirect/Get (PRG)**: after a successful POST, respond with a redirect, not a rendered page. Otherwise a browser refresh resubmits the POST.
- **`redirect(obj)`** calls `obj.get_absolute_url()`, which is why you added it in Step 5.
- **`@require_POST`** (from `django.views.decorators.http`) makes a view return 405 for any other method. Mutations must never be reachable by GET.

## Build this

Full-page behavior only. HTMX branches come in Steps 9–11.

1. Create `notes/forms.py` with `NoteForm(ModelForm)`, `Meta.model = Note`, `Meta.fields = ["title", "body"]`.
2. Replace the stubs in `notes/views.py`:

| View           | Behavior                                                                                                                                                                                                 |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `note_list`    | Read `q`, `page` and `archived` from `request.GET`. `archived=1` uses the archived set, otherwise the active set. Apply `.search(q)`. Paginate at 20. Render with `page_obj`, `q` and the archived flag. |
| `note_detail`  | `get_object_or_404`, render.                                                                                                                                                                             |
| `note_create`  | GET renders an empty form. Valid POST saves, adds `messages.success`, redirects to the note. Invalid POST re-renders the bound form (200, errors visible).                                               |
| `note_edit`    | Same as create, with `instance=note` on both the GET and POST forms.                                                                                                                                     |
| `note_delete`  | `@require_POST`, `get_object_or_404`, delete, `messages.success`, redirect to `notes:list`.                                                                                                              |
| `note_pin`     | `@require_POST`, toggle `is_pinned`, plain `save()` for now, redirect to `notes:list`.                                                                                                                   |
| `note_archive` | `@require_POST`, toggle `is_archived`, plain `save()`, redirect to `notes:list`.                                                                                                                         |

The spec doesn't say where non-HTMX pin/archive redirect; the list is the sensible target. `update_fields` refinements come in Step 11.

3. Templates arrive in Step 8. To run the views now, create bare placeholder files `templates/notes/note_list.html`, `note_detail.html` and `note_form.html` (one line each, for example printing the object or `{{ form.as_p }}`). Step 8 replaces them.
4. Verify by hand:
   - `/` returns 200; `/?page=999` and `/?page=abc` don't crash.
   - `/?archived=1` and `/?q=word` change the results.
   - Valid create redirects to the note; invalid create (empty title) stays on the form.
   - GET on `/notes/1/delete/` returns 405; `/notes/99999/` returns 404.
5. Commit: `feat(001): note form and function-based views`.

## Paste me

- `notes/forms.py` and `notes/views.py`.
- Anything that misbehaved in step 4's checks.

I'll review the views for correctness and idiom (query structure, PRG, method restrictions) before Step 8.

## Check question

After a valid POST you `redirect(note)` rather than `render(...)`. What specific problem does that avoid?
