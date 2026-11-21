# LMS — Learning Management System

A MERN learning platform: instructors create courses and upload lectures, students enroll (free or paid via Razorpay), watch lectures with progress tracking, ask the instructor doubts, and chat with an AI tutor (Google Gemini).

```text
LMS-final copy 3/
├── client/   React + Vite frontend (Netlify)   → github.com/Khushipanu/lmsclient
└── server/   Express + MongoDB API (Render)    → github.com/Khushipanu/lmsserver
```

## Roles

| Role | Stored as | Can do |
| --- | --- | --- |
| Student | `role: "user"` | Browse, enroll, watch lectures, ask doubts, AI tutor |
| Instructor | `role: "admin"` | Everything above + create/edit/delete own courses, upload lectures, see enrolled students, reply to doubts |

Pick "Learn" or "Teach" on the sign-up page.

## Run locally

```bash
cd server && cp .env.example .env   # fill it in
npm install && npm run dev          # http://localhost:5000
npm run seed                        # optional: demo instructor + 4 courses
```

```bash
cd client && npm install
VITE_API_URL=http://localhost:5000 npm run dev   # http://localhost:5173
```

Demo instructor from the seed: `admin@demo.com` / `demoadmin123` (change or delete it in production).

## Production checklist (Render + Netlify)

After deploying, open **`https://<your-api>.onrender.com/health`**. It shows the database state, the allowed frontend origins and which services are configured, without exposing secrets. Every item below should show as configured there.

### 1. Render (server) environment variables

| Variable | Value |
| --- | --- |
| `NODE_ENV` | `production` |
| `MONGODB_URI` | Atlas connection string. In Atlas → Network Access, allow `0.0.0.0/0` |
| `JWT_SECRET_KEY`, `ACTIVATION_SECRET`, `FORGOT_SECRET_KEY` | long random strings |
| `CLIENT_URL` | `https://learningsytemclient.netlify.app` (no trailing slash; comma-separate several) |
| `BREVO_API_KEY`, `EMAIL_FROM` | see step 2 |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_CALLBACK_URL` | see step 3 |
| `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET` | test or live keys |
| `CLOUDINARY_URL` | see step 4 |
| `GEMINI_API_KEY` | from aistudio.google.com |

Render does **not** read `server/.env`. Every value has to be added in the Render dashboard.

### 2. Email (signup codes and password reset)

Render's free plan **blocks SMTP**, so Gmail/nodemailer cannot work there. The app sends email through Brevo's HTTPS API instead.

1. In Brevo → *SMTP & API → API Keys*, create a **new** key and make sure it is active. (The current key is rejected with "API Key is not enabled".) If Brevo asks you to activate your account or transactional sending, finish that first.
2. In Brevo → *Senders, Domains & Dedicated IPs → Senders*, add the address you use as `EMAIL_FROM` and click the verification link Brevo emails you.
3. For reliable inbox delivery, verify a domain you own and send from it. Mail sent through Brevo with a `@gmail.com` sender often lands in spam.
4. Optional stopgap only: `EMAIL_OTP_FALLBACK=true` shows the signup code on screen when email fails. Leave it **off** for real users, because anyone could then sign up with any email address.

Alternatives: `RESEND_API_KEY` (needs a verified domain) or `SMTP_*` (local development or paid hosting only).

### 3. Google sign-in

In Google Cloud Console → *APIs & Services → Credentials → OAuth client (Web application)*:

- **Authorized redirect URI:** `https://learningmanagementserver.onrender.com/auth/google/callback`. This must exactly match `GOOGLE_CALLBACK_URL` on Render.
- **Authorized JavaScript origin:** `https://learningsytemclient.netlify.app`
- *OAuth consent screen:* while it's in **Testing**, only the listed test users can sign in. Click **Publish app** to open sign-in to everyone.

### 4. File uploads (Cloudinary)

Render's disk is wiped on every deploy and restart, so uploaded course images and videos disappear. Create a free Cloudinary account and copy the "API environment variable" (`cloudinary://…`) into `CLOUDINARY_URL` on Render. New uploads then go to Cloudinary automatically.

### 5. Netlify (client)

- Build command `npm run build`, publish directory `dist`
- Env var `VITE_API_URL=https://learningmanagementserver.onrender.com`. If it's missing, the production build falls back to that URL.
- `VITE_RAZORPAY_KEY_ID` is optional; the server now sends its key during checkout.
- SPA redirects are already in `public/_redirects`.

### Render free-tier note

The free server sleeps after about 15 minutes idle, and the first request after that takes 30–60 seconds. The site now shows a "Waking up the server…" message during that wait instead of failing.
