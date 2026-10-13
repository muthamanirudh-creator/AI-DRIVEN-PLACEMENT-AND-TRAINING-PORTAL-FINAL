# Career Fit RAG "session expired" + Algorithm generator (Gemini/Groq) — pass 5

## 1. Career Fit Q&A (RAG) kept saying "Your session expired"
Root cause: the 15-minute access cookie lapsed while several requests were in flight (RAG status poll, hybrid status,
AI/ML status, storage sync). Each got a 401 and each called `/api/auth/refresh` with the SAME refresh cookie. The server
rotates the refresh token on every use, so the 2nd/3rd call looked like a stolen token -> the server revoked every
refresh token and cleared the cookies -> the user was really signed out.

Fixes
- `frontend/app-runtime.js` `refreshPortalSession()` is now single-flight (all callers share one refresh), reuses a refresh
  that just succeeded (4 s) and uses a Web Lock so several tabs take turns.
- `backend/src/routes/auth.js` `/refresh`: a token retired < 30 s ago (env `REFRESH_REUSE_GRACE_MS`, 0 disables) is treated
  as a benign concurrent refresh (fresh access cookie only, the winner's refresh cookie is not touched). Real replay after
  the window still revokes the whole token family. A lost race no longer clears the winner's cookies.
- Career Fit panel: stops polling every 12 s after an auth failure; error card now has **Retry** (refreshes the session and
  re-asks) and **Sign in again**.
- Cache token bumped to `gc-officialletter2-hrag-auth2` so browsers load the new script.

## 2. Algorithm generator -> Gemini / Groq only (no OpenAI)
- The server gateway only ever supported Groq + Gemini, but the screen preferred/labelled "OpenAI". Now it prefers
  **Gemini** (strict JSON mode) and falls back to **Groq**; the label and error texts no longer mention OpenAI/Claude keys.
- `callAiProvider` forwards a Gemini/Groq preference; the server whitelists it (anything else is dropped -> normal chain).
- Per-call timeout now scales with requested output (`aiTimeoutMs`): 8000-token animation JSON used to hit the fixed 20 s
  limit; it now gets up to ~52 s on Vercel / 110 s elsewhere.

No feature was removed. Admin must have `GEMINI_API_KEY` and/or `GROQ_API_KEY` set (Admin -> Settings -> AI Keys & Services).
