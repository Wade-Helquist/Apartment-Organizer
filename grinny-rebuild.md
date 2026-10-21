# Grinny — Mobile-First Firebase Rebuild

## Goal
Convert Grinny from a single-page artifact (HTML + artifact DB) into a proper mobile-first React app backed by Firebase. Keep the discovery engine logic exactly as-is; replace the storage layer and redesign the UI for phone-first use.

---

## Tech Stack
- **Vite** — build tool + dev server
- **React 18** — UI framework
- **Tailwind CSS v3** — styling (mobile-first utility classes)
- **Firebase v10 (modular SDK)**
  - Firestore — user data (facts, questions, discovers, thoughts)
  - Auth — Google sign-in
  - Hosting — deployment

---

## Project Setup

```bash
npm create vite@latest grinny -- --template react
cd grinny
npm install
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
npm install firebase
npm install -g firebase-tools
firebase login
firebase init hosting   # public dir: dist, SPA: yes
```

**tailwind.config.js**
```js
export default {
  content: ["./index.html", "./src/**/*.{js,jsx}"],
  theme: { extend: {} },
  plugins: [],
}
```

**src/index.css** — add at top:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

---

## Firebase Setup (console.firebase.google.com)

1. Create project → name it `grinny`
2. Firestore → Create database → production mode → us-central1
3. Authentication → Get started → Google → enable → save
4. Project settings → Your apps → register web app → copy `firebaseConfig`
5. Firestore → Rules → publish this:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth.uid == userId;
    }
    match /shared/{document=**} {
      allow read: if request.auth != null;
      allow write: if false;
    }
  }
}
```

---

## File Structure

```
src/
  lib/
    firebase.js        # init + exports
  hooks/
    useAuth.js         # Google sign-in state
    useFacts.js        # Firestore CRUD for facts
    useDiscovers.js    # discovers collection
    useThoughts.js     # thoughts (answers) collection
  components/
    AuthGate.jsx       # show sign-in screen if not authed
    BottomNav.jsx      # mobile nav bar
    DiscoverPane.jsx   # main discovery UI (port from current app)
    FactCard.jsx       # single fact display
    FactList.jsx       # keyword → facts list
    FilterBar.jsx      # type filter + sort dropdowns
    HistoryPane.jsx    # history list
    QuestionsPane.jsx  # saved questions
    AddFactModal.jsx   # add/edit fact sheet
  App.jsx
  main.jsx
```

---

## src/lib/firebase.js

```js
import { initializeApp } from "firebase/app";
import { getFirestore } from "firebase/firestore";
import { getAuth, GoogleAuthProvider } from "firebase/auth";

const firebaseConfig = {
  // paste your config here
};

const app = initializeApp(firebaseConfig);
export const db = getFirestore(app);
export const auth = getAuth(app);
export const googleProvider = new GoogleAuthProvider();
```

---

## Data Model (Firestore)

All data lives under `users/{uid}/` — completely private per user.

```
users/{uid}/
  facts/{factId}        # same shape as current: s, p, o, i, tp, b, cat, cats[], from[], fav, hide, t
  thoughts/{thoughtId}  # answers: q, f[], m[], pend, ck, t, fav, mini
  questions/{questionId}# q, kind, from, t, fav
  discovers/{discoverId}# w, r, t, fav
```

**Starter pack** — optional shared collection the app seeds from on first sign-in:
```
shared/
  facts/{factId}        # read-only starter facts (Firestore rules: read=authed, write=false)
```

On first sign-in, copy `shared/facts` into `users/{uid}/facts` so each user starts with something.

---

## Porting the Discovery Engine

The current `grinny-export.html` contains all the logic. The key functions to extract into React hooks/utils:

| Current function | Where it goes |
|---|---|
| `gen()`, `sample` | `src/lib/ai.js` — Claude/OpenAI API calls |
| `addFact()`, `allFacts()` | `src/hooks/useFacts.js` |
| `dRender()`, `dCache` | `src/components/DiscoverPane.jsx` |
| `ddRun()`, `openDD()`, `dds` | `src/components/DiscoverPane.jsx` |
| `factIndex()`, `fcount()` | `src/hooks/useFacts.js` |
| `canon()`, `norm()`, `segs()` | `src/lib/utils.js` |
| `FT`, `FT_SHORT`, `FT_SKIP` | `src/lib/constants.js` |
| `autoCat()`, `aiCatSweep()` | `src/lib/categorize.js` |
| `mark()`, `wordsOf()` | `src/lib/markup.js` |
| `renderHistory()` | `src/components/HistoryPane.jsx` |
| `libRender()` | `src/components/QuestionsPane.jsx` |
| `factsRender()` | `src/components/FactList.jsx` |

**Replace the artifact DB calls** (`db.doc("facts/"+id).set(doc)`) with Firestore:
```js
import { doc, setDoc, collection, onSnapshot } from "firebase/firestore";
import { db } from "../lib/firebase";

// write a fact
await setDoc(doc(db, "users", uid, "facts", factId), factData);

// live listener (replaces polling)
onSnapshot(collection(db, "users", uid, "facts"), snapshot => {
  const facts = {};
  snapshot.forEach(d => facts[d.id] = d.data());
  setFacts(facts);
});
```

---

## Mobile UI Design

### Layout
- Full-screen single column, no sidebar
- **Bottom nav bar** (fixed, thumb-friendly, 5 tabs):
  - 🔍 Discover
  - 📚 Facts
  - 💬 Questions
  - 🕑 History
  - ⚙️ Tools

### Discover Pane (mobile)
- Search bar at top (sticky)
- Answer cards full width, large text
- Tappable words open a bottom sheet (replace the current inline dropdown)
- Bottom sheet slides up with the word's facts and related topics
- Swipe down to dismiss

### Key Tailwind patterns
```jsx
// Bottom nav
<nav className="fixed bottom-0 inset-x-0 bg-white dark:bg-gray-900 border-t border-gray-200 dark:border-gray-800 flex safe-pb">
  {tabs.map(tab => <NavButton key={tab.id} {...tab} />)}
</nav>

// Main content — pad for bottom nav
<main className="pb-20 px-4 max-w-xl mx-auto">

// Card
<div className="bg-white dark:bg-gray-900 rounded-2xl border border-gray-200 dark:border-gray-800 p-4 mb-3">

// Bottom sheet
<div className="fixed inset-x-0 bottom-0 z-50 bg-white dark:bg-gray-900 rounded-t-2xl p-4 shadow-2xl max-h-[80vh] overflow-y-auto">
```

### Safe area (iPhone notch/home bar)
Add to `index.html` viewport meta:
```html
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```
Use `pb-safe` / `env(safe-area-inset-bottom)` for bottom nav.

---

## Auth Flow

```jsx
// src/hooks/useAuth.js
import { signInWithPopup, signOut, onAuthStateChanged } from "firebase/auth";
import { auth, googleProvider } from "../lib/firebase";

export function useAuth() {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  useEffect(() => onAuthStateChanged(auth, u => { setUser(u); setLoading(false); }), []);
  const signIn = () => signInWithPopup(auth, googleProvider);
  const signOut_ = () => signOut(auth);
  return { user, loading, signIn, signOut: signOut_ };
}
```

```jsx
// src/components/AuthGate.jsx
export function AuthGate({ children }) {
  const { user, loading, signIn } = useAuth();
  if (loading) return <div className="flex items-center justify-center h-screen">Loading…</div>;
  if (!user) return (
    <div className="flex flex-col items-center justify-center h-screen gap-6 px-8">
      <h1 className="text-4xl font-bold">Grinny</h1>
      <p className="text-gray-500 text-center">Discoveries until you acquire</p>
      <button onClick={signIn} className="bg-blue-600 text-white rounded-2xl px-8 py-4 text-lg font-semibold w-full max-w-xs">
        Sign in with Google
      </button>
    </div>
  );
  return children;
}
```

---

## Deploy

```bash
npm run build
firebase deploy
```

Live at `https://grinny.web.app` (or your custom domain).

**Custom domain:** Firebase Hosting → Add custom domain → follow DNS steps.

---

## Suggested Build Order

1. Scaffold Vite + React + Tailwind + Firebase, get auth working
2. Wire Firestore — load/save facts, live listener
3. Port `DiscoverPane` — search, answer cards, tappable words
4. Port `FactList` — keyword browser with filter/sort
5. Add bottom nav, bottom sheet for word drill-down
6. Port history + questions panes
7. Test on real phone (Chrome DevTools → toggle device toolbar during dev)
8. `firebase deploy`

---

## Notes

- The AI calls (`gen()`, `sample`) use the artifact runtime's built-in Claude access. In the standalone app you will need to proxy these through a Firebase Cloud Function or call the Anthropic API directly with your own key.
- Keep `FT`, `FT_SHORT`, `canon()`, `norm()` exactly as-is — the discovery engine depends on them.
- The `qid()` hash function must stay identical so existing fact IDs stay stable if you migrate data.
