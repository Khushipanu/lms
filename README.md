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
| `GMAIL_SCRIPT_URL`, `GMAIL_SCRIPT_SECRET` | see step 2 |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_CALLBACK_URL` | see step 3 |
| `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET` | test or live keys |
| `CLOUDINARY_URL` | see step 4 |
| `GEMINI_API_KEY` | from aistudio.google.com |

Render does **not** read `server/.env`. Every value has to be added in the Render dashboard.

### 2. Email (signup codes and password reset)

Render's free plan **blocks SMTP**, so Gmail through nodemailer cannot work there. Brevo and Resend need a verified sender or domain. The app therefore sends through **your own Gmail via a Google Apps Script**, called over normal HTTPS, so no SMTP is involved.

1. Open [script.google.com](https://script.google.com), signed in to the Gmail account that should send the emails, and click **New project**.
2. Paste the contents of [`server/scripts/gmail-relay.gs`](server/scripts/gmail-relay.gs), then set `SECRET` to a long random string (letters and numbers only).
3. Click **Deploy → New deployment → Web app**. Set *Execute as* to **Me** and *Who has access* to **Anyone**, click **Deploy**, then click **Authorize access** and allow sending email. If Google says the app isn't verified, click *Advanced → Go to … (unsafe)*. It's your own script, so this warning is expected.
4. Copy the Web app URL (it ends in `/exec`). Opening it in a browser should show `{"ok":true,...}`.
5. On Render, set `GMAIL_SCRIPT_URL` to that URL and `GMAIL_SCRIPT_SECRET` to the same secret.

Limit: about 100 emails a day on a free Gmail account. Brevo or Resend still work as automatic backups if they're configured: every provider that's set is tried in order until one succeeds. `EMAIL_OTP_FALLBACK=true` (show the code on screen) is an emergency-only option.

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
