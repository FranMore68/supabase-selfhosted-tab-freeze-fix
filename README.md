# Fixing Supabase Tab Freeze in Self-Hosted Environments (GoTrue) 🚀

If you are running Supabase in a self-hosted environment (e.g., Coolify, Docker) and your frontend application completely freezes or hangs in an infinite redirect loop when switching tabs, you are likely hitting an architectural issue with `GoTrue` cold starts.

## ⚠️ The Problem

Supabase's GoTrue container can take between 5 to 15 seconds to respond after a cold start. If your frontend application attempts to validate the user session synchronously when the tab gains focus (for example, by calling `getSession()` inside a `visibilitychange` event or heavily relying on React Query's `refetchOnWindowFocus`), the network thread gets blocked waiting for GoTrue.

This causes:

- The browser tab to freeze.
- Redux / React Query hydration loops.
- Infinite redirects if your middleware fails to obtain a timely response.

## 🛠️ The Solution

To properly handle Supabase auth in self-hosted environments across tab switches, apply these three architectural rules:

### 1. Middleware Must Be Non-Blocking

**Never** instantiate or call Supabase client methods like `getUser()` or `getSession()` within your Edge middleware. Your middleware should strictly perform a simple cookie-check (e.g., verifying `sb-*` cookies exist) and let the client or server decide the actual validity later.

### 2. Do Not Block `visibilitychange`

Remove any `await supabase.auth.getSession()` or similar blocking calls inside your client-side event listeners. Instead, leverage `startAutoRefresh()` and `stopAutoRefresh()`.

```javascript
// ✅ CORRECT IMPLEMENTATION (Non-blocking)
document.addEventListener("visibilitychange", () => {
  if (document.visibilityState === "visible") {
    supabase.auth.startAutoRefresh();
  } else {
    supabase.auth.stopAutoRefresh();
  }
});

// ❌ INCORRECT (Freezes the UI)
document.addEventListener("visibilitychange", async () => {
  if (document.visibilityState === "visible") {
    await supabase.auth.getSession(); // <- THIS CAUSES TAB FREEZES
  }
});
```

### 3. Client Configuration

Ensure your instantiated browser client has `autoRefreshToken` enabled and `persistSession` enabled.
Also, ignore the `SIGNED_IN` event and only rely on the `INITIAL_SESSION` event to trigger any heavy user-data fetching, to gracefully wait for GoTrue's cold start.

---

_This architectural fix resolves the vast majority of tab-switching freezes in standard Next.js / React applications using self-hosted Supabase._

---

## 🤖 Antigravity Skill Included

If you use the Antigravity Agent ecosystem, you can add this directly to your `.agent/skills` folder. See the attached [`fixing-supabase-tab-freeze.md`](./fixing-supabase-tab-freeze.md) file.
