# E-Commerce

A full-stack storefront: React 19 + Vite frontend, Express 5 + MongoDB backend, with **Firebase Authentication** handled client-side and **verified server-side** on every protected request.

## Structure

```
frontend/   React 19 + Vite 7 + Tailwind 4 SPA
backend/    Express 5 + Mongoose API, Firebase Admin token verification
```

## How authentication actually works

This is the part worth reading. The client signs users in with Firebase and attaches the resulting ID token as a bearer header. The backend never trusts a user ID from the request body — `middleware/verifyToken.js` calls `admin.auth().verifyIdToken(token)` on every protected route and attaches the resolved identity:

```js
const decodedToken = await admin.auth().verifyIdToken(token);
req.user = { firebaseUid: decodedToken.uid, email: decodedToken.email };
```

The practical effect: **no passwords are stored or handled anywhere in this codebase**, and a forged token fails cryptographically rather than by a role check.

## What it does

- Product catalogue with admin-managed listings
- Cart that persists per user
- Reviews and star ratings
- Order and delivery tracking, with an automatic status transition
- Animated page transitions (GSAP), carousel (Swiper), toast and modal feedback

## The background job

`backend/app.js` registers a nightly cron that closes out deliveries whose expected date has passed:

```js
cron.schedule("0 0 * * *", async () => {
  await User.updateMany(
    { "orderedProd.expectedDeliveryDate": { $lte: today } },
    { $set: { "orderedProd.$[elem].isDelivered": true } },
    { arrayFilters: [{ "elem.expectedDeliveryDate": { $lte: today } }] }
  );
});
```

Using `arrayFilters` matters here — it updates only the matching elements inside the user's order array instead of rewriting every order the user has.

## API surface

| Base path | Scope |
|---|---|
| `/api/prod` | product catalogue |
| `/api/user` | profile and admin operations |
| `/api/cart` | cart contents |
| `/api/book` | booking / order handling |

## Stack

**Frontend** — React 19 · Vite 7 · Tailwind CSS 4 · React Router 7 · Axios · Firebase · GSAP · Swiper
**Backend** — Express 5 · Mongoose 8 · MongoDB · firebase-admin · node-cron · Twilio

## Configuration

Both sides ship an `.env.example`. Copy it, fill it, and you're running:

| File | Variables |
|---|---|
| `backend/.env.example` | `MONGO_URI`, `PORT`, `REACT_SERVER` (CORS origin) |
| `frontend/.env.example` | the seven `VITE_FIREBASE_*` client values |

Firebase Admin credentials are not environment variables — download a service-account key and save it as `backend/serviceAccountKey.json`, which is already gitignored.

The `.gitignore` excludes `.env*` broadly, so both templates are re-included explicitly with `!.env.example` at the end of the file.

## Run it

Backend:

```bash
cd backend
npm install
# .env needs MONGO_URI, FIREBASE_* service credentials, and optionally PORT
node app.js
```

Frontend:

```bash
cd frontend
npm install
npm run dev
```

## Known gaps

- `cors({ origin: process.env.REACT_SERVER || true })` falls back to `true`, which reflects **any** origin. With `credentials: true` that is too permissive for production — it should fail closed when the env var is missing.
- No tests. `npm test` is still the npm placeholder.

## License

MIT — see [LICENSE](./LICENSE).
