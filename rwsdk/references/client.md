# Client APIs

Import client helpers from `rwsdk/client`.

## initClient

`initClient()` hydrates the RSC flight payload appended to the page so client components become interactive.

```tsx
import { initClient } from "rwsdk/client";

initClient();
```

Handle action responses when you need custom redirect or status behavior:

```tsx
initClient({
  onActionResponse: (response) => {
    if (response.status === 401) {
      window.location.href = "/login";
      return true;
    }
  },
});
```

Return `true` to prevent default redirect handling.

## initClientNavigation

`initClientNavigation()` enables SPA-style client navigation for relative links. It intercepts document clicks, fetches the RSC payload for the new page, hydrates it, and updates browser history.

```tsx
import { initClientNavigation } from "rwsdk/client";

initClientNavigation();
```

Options:

```tsx
initClientNavigation({
  scrollToTop: true,
  scrollBehavior: "instant",
  onNavigate: async () => {
    await analytics.track(window.location.pathname);
  },
});
```

- `scrollToTop`: default `true`; set `false` to preserve scroll position.
- `scrollBehavior`: `"instant"`, `"smooth"`, or `"auto"`; default `"instant"`.
- `onNavigate`: runs after history update and before fetching the new RSC payload.

## navigate

Use `navigate()` for programmatic client navigation.

```tsx
import { navigate } from "rwsdk/client";

navigate("/about");
navigate("/profile", { history: "replace" });
navigate("/dashboard", {
  info: {
    scrollToTop: true,
    scrollBehavior: "smooth",
  },
});
```

Options:

- `history`: `"push"` default, or `"replace"`.
- `info.scrollToTop`: default `true`.
- `info.scrollBehavior`: `"instant"`, `"smooth"`, or `"auto"`.
