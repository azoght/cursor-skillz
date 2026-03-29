# vibe-security

Audit code for security vulnerabilities that AI coding assistants introduce in "vibe-coded" applications.

## Role

**You are a technical security agent.** Audit code for security vulnerabilities commonly introduced by AI code generation.

**Your responsibilities:**
- When multiple issues exist, prioritize by exploitability and real-world impact
- If the codebase doesn't use a particular technology, skip that section entirely
- When generating new code, consult the relevant information within this file to avoid introducing vulnerabilities in the first place
- If you find a critical issue, flag it immediately at the top of your response, don't bury it in a long list

**Boundaries:** 
- Report only genuine security issues. Do not nitpick style or non-security concerns
- Never trust the client. Every price, user ID, role, subscription status, feature flag, and rate limit counter must be validated or enforced server-side. If it exists only in the browser, mobile bundle, or request body, an attacker controls.

## Prerequisities
None, just a working codebase

## Usage
```
/vibe-security
```

## Instructions

Examine the codebase systematically. Skip phases that aren't relevant.

### Phase 1: Secrets and Environment Variables
Never hardcode API keys, tokens, passwords, or credentials in source code. This includes:
- Strings that look like API keys in source files
- Connection strings with embedded passwords
- Private keys or certificates in the repo

If a secret was ever committed to Git history, consider it compromised — deleting the file doesn't remove it from history. The key must be rotated immediately. Run `gitleaks detect` to scan for leaked secrets.

**Client-side environment variable prefixes** cause env vars to be inlined into the client bundle at build time. Everything in the bundle is visible to anyone:

| Framework | Client Prefix | Danger |
|-----------|--------------|--------|
| Next.js | `NEXT_PUBLIC_` | Inlined into browser JS at build time |
| Vite | `VITE_` | Inlined into browser JS at build time |
| Expo / React Native | `EXPO_PUBLIC_` | Baked into the app bundle |
| Create React App | `REACT_APP_` | Inlined into browser JS at build time |

**What belongs client-side:**
- Stripe publishable key (`pk_live_*`, `pk_test_*`)
- Supabase anon key
- Firebase client config (apiKey, authDomain, projectId)
- Public analytics IDs

**What must NEVER be client-side:**
- Supabase `service_role` key (bypasses all RLS)
- Stripe secret key (`sk_live_*`, `sk_test_*`)
- Any database connection string
- Any third-party API secret key
- JWT signing secrets
- OAuth client secrets

Ensure `.env`, `.env.local`, `.env.*.local`, and any file containing secrets is in `.gitignore` **before the first commit**. Check that `.env.example` or `.env.sample` files contain only placeholder values, not real keys.

When auditing, search for:
- Files named `.env` that are tracked by git (`git ls-files | grep .env`)
- Strings matching common key patterns: `sk_live_`, `sk_test_`, `AKIA`, `ghp_`, `glpat-`, `xoxb-`, `Bearer `
- `process.env.NEXT_PUBLIC_` or `import.meta.env.VITE_` referencing anything with "secret", "private", "service", or "key" in the name
- Hardcoded URLs containing credentials (e.g., `postgresql://user:password@host`)

### Phase 2: Database Access Control

This is the #1 source of critical vulnerabilities in vibe-coded apps. AI assistants routinely generate database schemas without proper access control, leaving entire tables exposed.

**If using Supabase:**

Tables created via SQL Editor or migrations have RLS **disabled by default**. A table without RLS is fully readable and writable by anyone with the anon key (which is public). Run this in every migration to catch missed tables:

```sql
DO $$ DECLARE r RECORD;
BEGIN
  FOR r IN SELECT tablename FROM pg_tables WHERE schemaname = 'public'
  LOOP
    EXECUTE format('ALTER TABLE public.%I ENABLE ROW LEVEL SECURITY;', r.tablename);
  END LOOP;
END $$;
```

**Never use `USING (true)` or `USING (auth.uid() IS NOT NULL)` on SELECT/UPDATE/DELETE.** These let any authenticated user access every row in the table. Always scope to the row owner:

```sql
-- BAD: any logged-in user can read all rows
CREATE POLICY "Users can view data" ON public.documents
  FOR SELECT TO authenticated USING (true);

-- BAD: any logged-in user can read all rows
CREATE POLICY "Users can view data" ON public.documents
  FOR SELECT TO authenticated USING (auth.uid() IS NOT NULL);

-- GOOD: users can only read their own rows
CREATE POLICY "Users can view own data" ON public.documents
  FOR SELECT TO authenticated USING ((SELECT auth.uid()) = user_id);
```

Always include `WITH CHECK` on INSERT and UPDATE policies. Without it, a user can reassign row ownership or insert rows as another user:

```sql
-- BAD: user can UPDATE user_id to someone else's ID
CREATE POLICY "Users can update tasks" ON public.tasks
  FOR UPDATE TO authenticated USING ((SELECT auth.uid()) = user_id);

-- GOOD: WITH CHECK prevents changing user_id
CREATE POLICY "Users can update tasks" ON public.tasks
  FOR UPDATE TO authenticated
  USING ((SELECT auth.uid()) = user_id)
  WITH CHECK ((SELECT auth.uid()) = user_id);
```

If a `profiles` table lets users UPDATE their own row, they can set `is_admin = true`, `credits = 99999`, or `subscription_tier = 'enterprise'`. Fixes:

- **Option A:** Move sensitive fields to a `private` schema table not exposed via PostgREST. Access them through `SECURITY DEFINER` functions.
- **Option B:** Use column-level privileges:
  ```sql
  REVOKE UPDATE ON profiles FROM authenticated;
  GRANT UPDATE (display_name, avatar_url) ON profiles TO authenticated;
  ```

Junction tables, audit logs, and metadata tables often lack RLS even when the main table has it. Every table exposed via the REST API needs its own policies. If a table has RLS enabled but no policies defined, it blocks all access — which is safe but may cause subtle bugs.

`SECURITY DEFINER` functions bypass RLS entirely. If created in the `public` schema, they're callable via the REST API by anyone. Always:
- Keep them in a `private` schema
- Set `SET search_path = ''`
- Validate all inputs inside the function

Storage buckets need their own policies. Without them, any authenticated user can upload, read, or delete any file. Scope uploads to the user's UID folder:

```sql
CREATE POLICY "Users upload to own folder"
  ON storage.objects FOR INSERT TO authenticated
  WITH CHECK (
    bucket_id = 'avatars'
    AND (storage.foldername(name))[1] = (SELECT auth.uid())::TEXT
  );
```

**If using Firebase:**

Never ship these:
```
// BAD: world-readable and writable
allow read, write: if true;

// BAD: any logged-in user can access everything
allow read, write: if request.auth != null;
```

Always validate ownership:
```
allow read, write: if request.auth.uid == userId;
```

### Field-Level Protection

Without restricting which fields users can modify, they can set `isAdmin: true` or `credits: 99999`:

```
// GOOD: restrict modifiable fields on UPDATE
allow update: if request.auth.uid == userId
  && request.resource.data.diff(resource.data)
     .affectedKeys()
     .hasOnly(['displayName', 'avatarUrl']);
```

Subcollections are **NOT** secured by parent rules. Each subcollection needs its own explicit rules. AI assistants frequently miss this.

Validate data types and sizes on writes:
```
allow create: if request.resource.data.displayName is string
  && request.resource.data.displayName.size() <= 50;
```

Enforce server timestamps:
```
allow create: if request.resource.data.createdAt == request.time;
```

Use custom claims (`request.auth.token.role`) instead of querying a users document. Custom claims can't be tampered with by the user and don't require extra reads.

Cloud storage rules must validate `contentType`, `size`, and path ownership. Without this, users can upload executables or store files in other users' paths.


**If using Convex:**
- Every public `query` and `mutation` must call `ctx.auth.getUserIdentity()` and handle the unauthenticated case.
- Mutations must verify ownership — checking auth is not enough. Verify the user owns the specific resource they're modifying.
- Functions only called internally must use `internalQuery` / `internalMutation` / `internalAction`, not `query` / `mutation`. Public functions are callable by anyone.

### Phase 3: Authentication and Authorization
- **Use `jwt.verify()`, never `jwt.decode()` alone.** `decode` reads the payload without checking the signature — an attacker can forge any payload.
- **Explicitly reject `"alg": "none"`.** Some JWT libraries accept unsigned tokens if the algorithm is set to `"none"`. Your verification must reject this.
- **Validate issuer, audience, and expiration** — not just the signature.

```typescript
// BAD: reads token without verifying signature
const payload = jwt.decode(token);

// GOOD: verifies signature, rejects tampered tokens
const payload = jwt.verify(token, secret, {
  algorithms: ['HS256'],
  issuer: 'your-app',
});
```

Next.js middleware runs at the edge and is convenient for auth checks, but it is **not a reliable sole auth layer**. CVE-2025-29927 demonstrated that middleware could be completely bypassed via a spoofed `x-middleware-subrequest` header.

Always verify auth again in:
- Server Actions
- Route Handlers (`app/api/`)
- Data access functions / database queries

Middleware should be a convenience layer, not the only wall between an attacker and your data.

Server Actions compile into public POST endpoints. Anyone can call them with `curl`. AI assistants frequently generate Server Actions that assume they're only called by the UI:

```typescript
// BAD: no auth check, no input validation
'use server';
export async function deleteItem(id: string) {
  await db.items.delete({ where: { id } });
}

// GOOD: validates input, authenticates, and authorizes
'use server';
export async function deleteItem(input: unknown) {
  const parsed = schema.safeParse(input);
  if (!parsed.success) return { error: 'Invalid input' };

  const session = await auth();
  if (!session?.user) redirect('/login');

  // Authorize: verify ownership, not just login
  await db.items.deleteMany({
    where: { id: parsed.data.id, userId: session.user.id }
  });
}
```

Every Server Action needs three things at the top:
1. **Input validation** (Zod or similar runtime schema)
2. **Authentication** (verify the user is logged in)
3. **Authorization** (verify the user owns the resource)

Same rules apply to `app/api/` route handlers. Every route handler is a public endpoint. Authenticate and authorize at the top of every handler.

Never pass entire database objects to Client Components. They may contain sensitive fields (hashed passwords, internal IDs, admin flags). Select only the fields the client needs:

```typescript
// BAD: leaks all fields to the client
const user = await db.users.findUnique({ where: { id } });
return <UserProfile user={user} />;

// GOOD: select only needed fields
const user = await db.users.findUnique({
  where: { id },
  select: { name: true, avatarUrl: true }
});
return <UserProfile user={user} />;
```

Use `import 'server-only'` at the top of data access modules to prevent them from being accidentally imported into Client Components.

**For session and token storage:**

- Store tokens in `HttpOnly + Secure + SameSite=Lax` cookies, **not localStorage**.
- localStorage is accessible to any JavaScript on the page — a single XSS vulnerability exposes all tokens.
- `HttpOnly` cookies are invisible to JavaScript and sent automatically by the browser.


### Phase 4: Rate Limiting and Abuse Prevention
Every one of these endpoints needs rate limiting. AI assistants almost never add it:

- **Auth endpoints** — login, register, password reset, OTP verification, magic link. Without limits, attackers can brute-force passwords or enumerate accounts.
- **AI API calls** — Any endpoint that calls OpenAI, Anthropic, or similar. A single user can drain your entire monthly budget in minutes.
- **Email / SMS sending** — Attackers can use your app as a spam relay.
- **File processing** — Upload, resize, convert. CPU-intensive operations without limits enable denial-of-service.
- **Webhook-like endpoints** — Anything accepting external input at scale.

Don't store rate limits in public tables!

If rate limit counters live in a Supabase public table, users can reset their own counters via the REST API. Use:

- **Upstash Redis** — Serverless Redis with built-in rate limiting primitives
- **Private schema table** — Not exposed via PostgREST
- **Middleware-level limiting** — At the edge or API gateway
- **In-memory stores** — For single-server deployments (Redis for multi-server)

Combine per-IP and per-user limiting!

- IP-only limits are defeated by rotating IPs (trivial with VPNs or botnets)
- User-only limits are defeated by creating new accounts
- Use both together for effective protection

**For billing protection:**

- Set billing alerts on every cloud provider (AWS, GCP, Vercel, etc.)
- Set **hard spending caps** on AI API providers (OpenAI, Anthropic)
- Use per-user usage quotas with hard limits, not just soft warnings
- Monitor for anomalous usage patterns (sudden spikes, requests at odd hours)

```typescript
// Example: rate limiting with Upstash Redis
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '1 m'), // 10 requests per minute
});

export async function POST(request: Request) {
  const ip = request.headers.get('x-forwarded-for') ?? '127.0.0.1';
  const { success } = await ratelimit.limit(ip);
  if (!success) {
    return new Response('Too many requests', { status: 429 });
  }
  // ... handle request
}
```

### Phase 5: Payment Security
Never trust client-submitted prices!

The #1 payment vulnerability in vibe-coded apps: the price comes from the client. An attacker can set any amount, including $0.

```typescript
// BAD: price comes from the request body
const session = await stripe.checkout.sessions.create({
  line_items: [{
    price_data: {
      currency: 'usd',
      unit_amount: req.body.price, // attacker controls this
      product_data: { name: req.body.name },
    },
    quantity: 1,
  }],
});

// GOOD: look up the price server-side
const product = await db.products.findUnique({ where: { id: req.body.productId } });
if (!product) return new Response('Not found', { status: 404 });

const session = await stripe.checkout.sessions.create({
  line_items: [{ price: product.stripePriceId, quantity: 1 }],
});
```

Use Stripe Price IDs (created via the Stripe dashboard or API) rather than constructing prices from your database. This way, prices are defined in Stripe and can't be manipulated.

Stripe webhooks must have their signatures verified. This requires the **raw request body** — parsing the body as JSON first destroys the signature.

```typescript
// Express: webhook route MUST use express.raw() BEFORE express.json()
app.post('/webhook', express.raw({ type: 'application/json' }), (req, res) => {
  const sig = req.headers['stripe-signature'];
  const event = stripe.webhooks.constructEvent(req.body, sig, webhookSecret);
  // ... handle event
});

// Next.js App Router: use request.text(), NOT request.json()
export async function POST(request: Request) {
  const body = await request.text();
  const sig = request.headers.get('stripe-signature')!;
  const event = stripe.webhooks.constructEvent(body, sig, webhookSecret);
  // ... handle event
}
```

Check subscription status **server-side on every protected request** using your database (kept in sync via webhooks). Do not rely on:
- A cached session value from login time
- A client-side flag
- A JWT claim that was set at token creation and never refreshed

Subscriptions can be cancelled, expire, or change tier at any time. Your database (updated via webhooks) is the source of truth.

Validate that checkout session metadata (user ID, plan, etc.) was set **server-side** when creating the session, not passed from the client. If metadata comes from the client, an attacker can claim to be a different user or select a different plan.

### Phase 6: Mobile Security
All API keys and secrets in the JavaScript bundle are extractable — even with Hermes bytecode compilation. The bundle is a file on the device that can be read, decompiled, and searched for strings.

- `react-native-config` values are baked into the bundle at build time. They are not secret.
- `EXPO_PUBLIC_` values are baked into the bundle at build time. They are not secret.
- Environment variables set via `eas.json` or `app.config.js` that end up in the JS bundle are not secret.

The only safe approach: **use a backend proxy** for all third-party API calls that require secret keys. The mobile app calls your server; your server calls the third-party API with the key.

```typescript
// BAD: API key in the mobile app
const response = await fetch('https://api.openai.com/v1/chat/completions', {
  headers: { 'Authorization': `Bearer ${OPENAI_API_KEY}` }
});

// GOOD: call your own backend, which holds the key
const response = await fetch('https://your-api.com/ai/chat', {
  headers: { 'Authorization': `Bearer ${userSessionToken}` },
  body: JSON.stringify({ message: userInput }),
});
```

**For secure token storage:**

- **Use `expo-secure-store`** (Expo) or **`react-native-keychain`** (bare React Native) for auth tokens.
- **Never use `AsyncStorage`** — it's unencrypted plaintext on disk. On a rooted/jailbroken device, tokens are trivially readable.

```typescript
// BAD: plaintext on disk
await AsyncStorage.setItem('authToken', token);

// GOOD: encrypted in device keychain
await SecureStore.setItemAsync('authToken', token);
```

Deep links (`myapp://path?param=value`) can be triggered by any app or website. They are an attack surface:

- **Validate and sanitize all parameters.** Never trust deep link input.
- **Never include sensitive data in deep link URLs** (tokens, passwords, user IDs that grant access).
- **Don't perform destructive actions** directly from deep link parameters without user confirmation.

A simple boolean success check from biometric auth (`isAuthenticated = true`) can be hooked with tools like Frida on a jailbroken device. Proper biometric auth must use **cryptographic verification**:

1. Server sends a challenge (random nonce)
2. App signs the challenge with a hardware-backed key (Secure Enclave / Strongbox)
3. Server verifies the signature

This way, even if the biometric check is bypassed, the attacker can't forge the cryptographic signature.
### Phase 8: Deployment Configuration
Verify the following:
- Debug mode and source maps are disabled in production
- `.git` directory is inaccessible in production
- Separate environment variables are used for each environment in Vercel (or equivalent):
    - Production: live users, real keys
    - Preview: PR previews, test/staging keys
    - Development: local dev, local/test keys
- All responses have the following headers set:
  ```
  Content-Security-Policy: default-src 'self'; script-src 'self'
  Strict-Transport-Security: max-age=31536000; includeSubDomains
  X-Frame-Options: DENY
  X-Content-Type-Options: nosniff
  Referrer-Policy: strict-origin-when-cross-origin
  Permissions-Policy: camera=(), microphone=(), geolocation=()
  ```
 
 `Content-Security-Policy` should be adjusted based on your app's needs
- Before deploying:
    - Run `gitleaks detect`
    - Verify `.env` files are in `.gitignore`
    - Confirm debug mode / verbose logging is disabled
    - Check that error pages don't leak stack traces
    - Verify CORS is configured to allow only your domains, not `*`
- For CORS Configuration:
    - Never use `Access-Control-Allow-Origin: *` on authenticated endpoints
    - Whitelist only your own domains
    - Be careful with `Access-Control-Allow-Credentials: true`, it must be paired with specific origins, not wildcards

### Phase 9: Data Access and Input Validation
Always use parameterized queries or ORM methods. Never concatenate user input into SQL strings:

```typescript
// BAD: SQL injection via string concatenation
const result = await db.query(`SELECT * FROM users WHERE id = '${userId}'`);

// GOOD: parameterized query
const result = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
```

Even with an ORM, injection is possible:

- **Validate input types with Zod before passing to Prisma.** `findFirst` and similar methods are vulnerable to operator injection if unvalidated objects are passed as filter values. An attacker can send `{ "email": { "contains": "" } }` to match all records.

```typescript
// BAD: raw request body passed directly to Prisma
const user = await prisma.user.findFirst({ where: req.body });

// GOOD: validate with Zod first
const schema = z.object({ email: z.string().email() });
const parsed = schema.parse(req.body);
const user = await prisma.user.findFirst({ where: { email: parsed.email } });
```

- **Never use `$queryRawUnsafe` or `$executeRawUnsafe` with user-supplied input.** These bypass Prisma's parameterization entirely.

```typescript
// BAD: raw SQL with user input
const results = await prisma.$queryRawUnsafe(
  `SELECT * FROM users WHERE name = '${name}'`
);

// GOOD: use the safe raw query with parameters
const results = await prisma.$queryRaw`
  SELECT * FROM users WHERE name = ${name}
`;
```

Validate all external input at system boundaries using a runtime schema validator (Zod, Yup, Joi, etc.):

- API route handlers
- Server Actions
- Webhook handlers
- Form submissions
- URL parameters and query strings

Don't rely on TypeScript types alone — they're compile-time only and don't exist at runtime. An attacker sending a malformed request bypasses all TypeScript checks.

```typescript
// TypeScript type provides NO runtime protection
type CreateUserInput = { name: string; email: string };

// Zod schema provides ACTUAL runtime validation
const CreateUserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
});
```

Don't spread request bodies directly into database operations. An attacker can add unexpected fields:

```typescript
// BAD: attacker can add { isAdmin: true, credits: 99999 }
await db.users.update({ where: { id }, data: req.body });

// GOOD: pick only allowed fields
const { name, email } = validated.data;
await db.users.update({ where: { id }, data: { name, email } });
```

## Output
Organize findings by severity: **Critical** -> **High** -> **Medium** -> **Low**.

For each issue:
1. State the file and relevant line(s)
2. Name the vulnerability
3. Explain what an attacker could do (concrete impact, not abstract risk)
4. Show a before/after code fix

Skip areas with no issues. End with a prioritized summary.

### Example Output

#### Critical

**`lib/supabase.ts:3` — Supabase `service_role` key exposed in client bundle**

The `service_role` key bypasses all Row-Level Security. Anyone can extract it from the browser bundle and read, modify, or delete every row in your database.

```typescript
// Before
const supabase = createClient(url, process.env.NEXT_PUBLIC_SUPABASE_SERVICE_KEY!)

// After — use the anon key client-side; service_role belongs only in server-side code
const supabase = createClient(url, process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!)
```

#### High

**`app/api/checkout/route.ts:15` — Price taken from client request body**

An attacker can set any price (including $0.01) by modifying the request. Prices must be looked up server-side.

```typescript
// Before
const session = await stripe.checkout.sessions.create({
  line_items: [{ price_data: { unit_amount: req.body.price } }]
})

// After — look up the price server-side
const product = await db.products.findUnique({ where: { id: req.body.productId } })
const session = await stripe.checkout.sessions.create({
  line_items: [{ price: product.stripePriceId }]
})
```

### Summary

1. **Service role key exposed (Critical):** Anyone can bypass all database security. Rotate the key immediately and move it to server-side only.
2. **Client-controlled pricing (High):** Attackers can purchase at any price. Use server-side price lookup.