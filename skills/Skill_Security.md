# Security Conventions

Patterns and conventions for security-related code in Datapath VCM.

## Sanitization

- **Rich text (TipTap)**: Always sanitize HTML with `sanitizeRichText()` from `lib/security/sanitize.ts` **before storing** to the database, not on read. Apply to any field that accepts HTML content (`key_terms`, `general_notes`).
- **Rendered HTML**: Use `sanitizeRenderedHtml()` or `DOMPurify.sanitize()` before any `dangerouslySetInnerHTML` usage.
- **Import**: `import { sanitizeRichText } from '@/lib/security/sanitize'`

## Rate Limiting

- Rate limiters are defined in `lib/security/rate-limit.ts` using `@upstash/ratelimit`
- Use `checkRateLimit(limiter, identifier)` — returns a 429 `NextResponse` or `null` if allowed
- Use `getClientIp(request)` for IP-based identifiers (handles `x-forwarded-for` behind Vercel proxy)
- Limiters: `authLimiter`, `registrationLimiter`, `aiLimiter`, `generalLimiter`
- Rate limiting is gracefully disabled when Upstash env vars are not configured (dev mode)

## Password Validation

- Always use the shared schema from `lib/validation/password.ts`
- Import: `import { passwordSchema } from '@/lib/validation/password'`
- Never define password rules inline — the shared schema ensures consistency (12+ chars, uppercase, lowercase, digit, special char)

## Authentication Patterns

- API routes: `const supabase = await createClient()` → `supabase.auth.getUser()` → check for `user`
- Service client bypasses RLS — only use in trusted server-side contexts (cron, admin actions)
- Never expose `SUPABASE_SERVICE_ROLE_KEY` to the client

## MFA / 2FA

- Supabase handles all TOTP factor storage and verification natively
- Enrollment: `supabase.auth.mfa.enroll({ factorType: 'totp' })`
- Challenge/verify: `supabase.auth.mfa.challenge({ factorId })` → `supabase.auth.mfa.verify({ factorId, challengeId, code })`
- AAL levels: `aal1` = password only, `aal2` = password + TOTP verified
- Per-user enforcement: `user_profiles.mfa_required` boolean — enforced in `proxy.ts`
- Admin reset: `POST /api/admin/users/[id]/reset-mfa` — uses `serviceClient.auth.admin.mfa.deleteFactor()`

## Secrets Management

- Never generate and display secrets in conversation — write directly to files
- Use `openssl rand -base64 32` (or shell equivalent) and pipe to the target file
- All secrets in `.env.local` (local) and Vercel Environment Variables (production)
- `.env*` is in `.gitignore` — never commit secrets

## Security Headers

- Defined in `next.config.ts` `headers()` function
- Applied to all routes via `source: '/(.*)'`
- CSP `connect-src` must include Supabase, Anthropic, and Slack domains

## Slack Webhook Verification

- Always use `crypto.timingSafeEqual()` for signature comparison — never `===`
- Verify timestamp is within 5-minute window to prevent replay attacks
- Implementation: `lib/slack/verify-signature.ts`

## Input Validation

- All API routes must validate input with Zod schemas
- Use `z.discriminatedUnion()` for endpoints that accept different action types
- Never use `body as { ... }` type assertions without Zod validation
