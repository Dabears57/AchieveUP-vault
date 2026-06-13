---
tags: [frontend, walkthrough, claude-created]
repo: achieveup-frontend
file: src/contexts/AuthContext.tsx
---

# AuthContext.tsx Walkthrough (`src/contexts/AuthContext.tsx`)

**The one-sentence job of this file:** it is the **auth system** for the entire app — it tracks who is logged in, handles login/logout/signup, and makes that information available to *any* component anywhere in the tree without passing props.

The file has 6 meaningful chunks:

| Lines | Chunk | What it is |
|---|---|---|
| 1–3 | Imports | React tools + the API layer + types |
| 5–14 | `AuthContextType` | The contract — what the context exposes |
| 16–24 | `AuthContext` + `useAuth` | The slot and the key |
| 30–86 | `AuthProvider` state + `checkAuthStatus` | The data and the startup check |
| 88–239 | `login` / `signup` / `logout` / `refreshUser` | The four actions |
| 241–256 | `value` assembly + Provider return | Wrapping the app |

---

## The Big Idea: Context

Before reading line-by-line, you need one concept: **Context is React's way of sharing data without prop drilling.**

Normally to pass data down a React tree you'd have to hand it through every component layer as a prop. With Context, you put data into a "store" at the top, and any component anywhere below can reach in and grab exactly what it needs. No passing required.

This file creates that store for authentication. Any component — a nav bar, a protected page, a button — can call `useAuth()` and immediately know if someone is logged in, who they are, and how to log them out. [[React Concepts]] has more on Context if you want the full mental model.

---

## Part 1: The Contract — `AuthContextType` (lines 5–14)

```ts
interface AuthContextType {
  user: User | null;
  loading: boolean;
  login: (email: string, password: string) => Promise<boolean>;
  signup: (data: SignupRequest) => Promise<boolean>;
  logout: () => void;
  refreshUser: () => Promise<void>;
  isAuthenticated: boolean;
  backendAvailable: boolean;
}
```

This TypeScript interface is the **public API of the auth system**. It lists every piece of data and every function that will be available to components. Think of it as a menu — consumers can order anything on it. The actual implementations come later; this just names the items.

- `user` — the logged-in user object, or `null` if nobody is logged in
- `loading` — `true` while the app is validating a stored token on startup
- `login` / `signup` / `logout` / `refreshUser` — the four auth actions
- `isAuthenticated` — a convenience boolean derived from `user` (see Part 6)
- `backendAvailable` — `false` if the Heroku server can't be reached; used to show an offline warning in the UI

---

## Part 2: The Slot and the Key (lines 16–24)

```ts
const AuthContext = createContext<AuthContextType | undefined>(undefined);
```

`createContext` creates the empty "slot." At startup it holds `undefined` — it only gets a real value once `AuthProvider` mounts and fills it. The `<AuthContextType | undefined>` tells TypeScript what the slot will eventually hold.

```ts
export const useAuth = () => {
  const context = useContext(AuthContext);
  if (context === undefined) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
};
```

`useAuth` is a **custom hook** — it's the handle every other component uses to reach into the context. Instead of writing `useContext(AuthContext)` in every file, components just call `useAuth()` and get back the full `AuthContextType` object.

The `undefined` check is a guard rail: if someone accidentally calls `useAuth()` outside the `AuthProvider` tree, they get a clear error message instead of a silent undefined-is-not-a-function crash.

**Key React idea — custom hooks.** Any function that starts with `use` and calls at least one built-in React hook is a "custom hook." It's just a regular function — the naming convention is a signal to React's linting tools that it follows hook rules. `useAuth` wraps `useContext`; that's all. See [[React Concepts]].

---

## Part 3: State — Three Variables (lines 31–33)

```ts
const [user, setUser] = useState<User | null>(null);
const [loading, setLoading] = useState(true);
const [backendAvailable, setBackendAvailable] = useState(true);
```

These three `useState` calls are the memory of the auth system. When any of them changes, React re-renders everything that consumes the context.

Notice `loading` starts as **`true`**, not `false`. That's intentional — the app assumes it's still checking auth until proven otherwise. This prevents a flash of the login screen on page refresh while the stored token is being validated (more on this in Part 4).

---

## Part 4: Checking Auth on App Load (lines 36–86)

```ts
const checkAuthStatus = async () => { ... }

useEffect(() => {
  checkAuthStatus();
}, []);
```

The `useEffect` with `[]` (empty dependency array) runs **once**, right after `AuthProvider` first renders. It calls `checkAuthStatus`, which is the startup routine:

```
1. Look in localStorage for a saved 'token'
2. No token → setUser(null), done (loading → false)
3. Token found → call authAPI.me()  (GET /achieveup/auth/me with Bearer token)
4. Check role on returned user object
   - instructor / admin / no Canvas token → setUser(userData), logged in
   - anything else (student) → clear token, setUser(null), logged out
5. Any error → clear token, setUser(null)
6. Network error specifically → also set backendAvailable = false
```

This is how the app **remembers you between page refreshes.** The JWT lives in `localStorage` (survives refresh), but it gets validated against the server every time so expired or tampered tokens are caught immediately. See [[JWT Authentication]] for what a JWT actually is and why this check matters.

**Key React idea — `useEffect`.** Effects are code that runs *after* React renders, in response to something changing. An empty `[]` dependency array means "run this effect only once, after the first render." See [[React Concepts]].

---

## Part 5: The Four Actions (lines 88–239)

### `login` (lines 88–151)

The flow:

```
authAPI.login(email, password)     → POST /achieveup/auth/login
  ↳ response.data.token            → localStorage.setItem('token', ...)
  ↳ authAPI.me()                   → GET /achieveup/auth/me (verify role)
  ↳ role check passes              → setUser(userData), return true
  ↳ role check fails               → clear token, return false
```

**Role gate (lines 106–116):** even after a successful login, the code checks if the user is an instructor. If not, it wipes the token and returns `false`. This is an instructor-only portal — students can't log in even if they have a valid account.

**Graceful fallback (lines 121–132):** if `authAPI.me()` fails *after* a successful login (network hiccup, etc.), login still returns `true`. The context builds a minimal user object from what it already knows rather than leaving the user stranded at a login screen despite having a valid token.

---

### `signup` (lines 153–210)

Same two-step pattern as `login`: call signup endpoint → get token → call `/auth/me` → setUser.

Key detail at lines 158–161: the signup payload is **hardcoded to `instructor` role** before the request goes out:

```ts
const signupData = {
  ...data,
  canvasTokenType: 'instructor' as const,
  role: 'instructor' as const
};
```

Whatever role the caller passed in gets overridden. You cannot create a student account through this portal, period.

---

### `logout` (lines 212–215)

```ts
const logout = () => {
  localStorage.removeItem('token');
  setUser(null);
};
```

Two lines. Token gone, user state gone, React re-renders the whole app — `isAuthenticated` becomes `false`, `ProtectedRoute` in [[App.tsx Walkthrough]] catches it and redirects to `/login`.

---

### `refreshUser` (lines 217–239)

Re-fetches the current user from the backend and updates state. Used when something changes server-side (like a user updates their Canvas token) and the in-memory user object needs to reflect that without a full logout/login cycle. If the fetch fails, it calls `logout()` to clean up.

---

## Part 6: Assembling and Providing (lines 241–256)

```ts
const value: AuthContextType = {
  user,
  loading,
  login,
  signup,
  logout,
  refreshUser,
  isAuthenticated: !!user,      // ← the only derived value
  backendAvailable,
};

return (
  <AuthContext.Provider value={value}>
    {children}
  </AuthContext.Provider>
);
```

This is where everything comes together. The `value` object is assembled from all the state and functions above, then handed to `<AuthContext.Provider>`.

**`!!user` — double-bang trick.** JavaScript's `!!` converts any value to a boolean: `!!null → false`, `!!{ ...userObject } → true`. So `isAuthenticated` is just a shorthand boolean for "is there a user object in state right now?" Components that only need to know *if* someone is logged in use `isAuthenticated`; components that need to know *who* use `user`.

**`children` — the wrapper pattern.** `AuthProvider` receives `children` as a prop. Whatever you wrap inside `<AuthProvider>` in JSX arrives here and gets rendered as-is. In `App.tsx`, the entire app is wrapped in `<AuthProvider>`, so the whole tree sits inside that `<AuthContext.Provider>` and can access the context.

---

## ⚠️ Quirks worth knowing

- **The flexible role check appears four times.** The same `isInstructor` logic (lines 52–55, 106–109, 224–226) is copy-pasted into `checkAuthStatus`, `login`, and `refreshUser`. It should ideally be a shared helper function. The comment says "allow users without Canvas token" — this is a deliberate loosening so instructors can sign up before adding their Canvas API token.

- **`loading` starts as `true`.** This is intentional (line 33), not a bug. The spinner in `ProtectedRoute` depends on it. Without it, every page refresh would show a flash of the login page before auth status resolves.

- **Network errors vs auth errors are treated differently.** An HTTP 401 (bad token) clears the token and sets `user = null` but leaves `backendAvailable = true`. A network-level failure (`NETWORK_ERROR` or `fetch`-related) additionally sets `backendAvailable = false`. The UI uses `backendAvailable` to decide whether to show "wrong password" vs "can't reach server."

- **There is no server-side logout call.** `logout()` only clears the local token — it does not hit a backend `/logout` endpoint. The JWT remains technically valid on the server until it expires on its own. This is common but worth knowing: there is no server-side token invalidation here.

- **The empty `AuthContext.tsx.md`** at the vault root is a leftover stub — this note replaces it.

---

## Check yourself (answer in your own notes)

1. A user navigates to `/dashboard` after their JWT expires overnight. Trace exactly what happens, naming every function that runs and what state it sets.
2. Why does `loading` start as `true` instead of `false`? What would a user see if it started as `false`?
3. What is the difference between `isAuthenticated` and `user`? Name a scenario where you'd want each one specifically.
4. If `authAPI.me()` fails immediately after a successful login, does the user end up logged in or logged out? Find the line that decides this.
5. Why does `logout()` not call the backend? What's the implication if you wanted to forcibly invalidate a stolen token?

**Next file:** [[Frontend API Layer]] covers `services/api.ts` — where `authAPI.login`, `authAPI.me`, and every other network call actually live.
