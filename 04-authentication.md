# 04 — Authentication

This guide sets up authentication with **NextAuth / Auth.js**: email + password, Google, and Microsoft. In single-tenant mode, anyone who logs in is a team member; the first user can be made an `ADMIN`.

## Install

```bash
npm install next-auth @auth/prisma-adapter bcryptjs
npm install -D @types/bcryptjs
```

## The auth config

Create `src/lib/auth.ts`:

```ts
import NextAuth from "next-auth";
import Credentials from "next-auth/providers/credentials";
import Google from "next-auth/providers/google";
import MicrosoftEntraId from "next-auth/providers/microsoft-entra-id";
import bcrypt from "bcryptjs";
import { prisma } from "@/lib/prisma";

export const { handlers, auth, signIn, signOut } = NextAuth({
  trustHost: true,
  session: { strategy: "jwt" },
  pages: { signIn: "/login" },
  providers: [
    Google({ allowDangerousEmailAccountLinking: true }),
    MicrosoftEntraId({
      clientId: process.env.MICROSOFT_CLIENT_ID!,
      clientSecret: process.env.MICROSOFT_CLIENT_SECRET!,
      issuer: `https://login.microsoftonline.com/${process.env.MICROSOFT_TENANT_ID}/v2.0`,
      allowDangerousEmailAccountLinking: true,
    }),
    Credentials({
      credentials: { email: {}, password: {} },
      async authorize(creds) {
        const user = await prisma.user.findUnique({ where: { email: String(creds?.email) } });
        if (!user?.passwordHash) return null;
        if (!user.emailVerified) throw new Error("EMAIL_NOT_VERIFIED");
        const ok = await bcrypt.compare(String(creds?.password), user.passwordHash);
        return ok ? { id: user.id, email: user.email, name: user.name } : null;
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) token.id = user.id;
      return token;
    },
    async session({ session, token }) {
      if (session.user) session.user.id = token.id as string;
      return session;
    },
    async signIn({ user, account }) {
      // For OAuth sign-ins, create the User row on first login.
      if (account?.provider !== "credentials" && user.email) {
        await prisma.user.upsert({
          where: { email: user.email },
          update: {},
          create: { email: user.email, name: user.name ?? user.email, emailVerified: new Date() },
        });
      }
      return true;
    },
  },
});
```

Expose the route handler:

```ts
// src/app/api/auth/[...nextauth]/route.ts
import { handlers } from "@/lib/auth";
export const { GET, POST } = handlers;
```

## Sign-up & email verification (credentials)

For email/password sign-up:

1. Hash the password with `bcrypt` and create the `User` with `emailVerified = null`.
2. Generate a one-time token, store it, and email a verification link.
3. On link click, set `emailVerified` and let them log in.

Keep verification tokens in a small table:

```prisma
model EmailVerificationToken {
  id        String   @id @default(uuid())
  userId    String
  token     String   @unique
  expiresAt DateTime
}
```

The same pattern (token table + emailed link) covers password reset.

## Protecting the dashboard

Use middleware to gate the dashboard and bounce logged-in users away from auth pages. Critically — **redirect `www` to the bare domain** so the session cookie (set on the apex domain) is always sent.

```ts
// src/middleware.ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

const protectedPrefixes = ["/dashboard", "/event-types", "/bookings", "/availability", "/routing", "/workflows", "/contacts", "/settings", "/integrations"];

export function middleware(req: NextRequest) {
  const { pathname } = req.nextUrl;

  // www → apex (avoids cookie-domain mismatch)
  const host = req.headers.get("host") || "";
  if (host.startsWith("www.")) {
    const url = new URL(req.url);
    url.host = host.replace("www.", "");
    return NextResponse.redirect(url, 301);
  }

  const token =
    req.cookies.get("__Secure-authjs.session-token") ??
    req.cookies.get("authjs.session-token");

  if (token && (pathname === "/login" || pathname === "/signup")) {
    return NextResponse.redirect(new URL("/dashboard", req.nextUrl.origin));
  }

  const isProtected = protectedPrefixes.some((p) => pathname.startsWith(p));
  if (isProtected && !token) {
    return NextResponse.redirect(new URL("/login", req.nextUrl.origin));
  }
  return NextResponse.next();
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|api|embed).*)"],
};
```

## OAuth provider setup

For each provider, register the **authorized redirect URI**:

- Google: `https://yourdomain.com/api/auth/callback/google`
- Microsoft: `https://yourdomain.com/api/auth/callback/microsoft-entra-id`

And for local dev, add the `http://localhost:3000/...` equivalents. Set `NEXTAUTH_URL` to your production URL.

> The calendar integration (next guide) needs the **same** Google/Microsoft accounts but with **additional scopes** (calendar read/write). Decide early whether login OAuth and calendar OAuth share one consent flow or are separate connect buttons. Keeping them separate (login = identity, "Connect Calendar" = calendar scopes) is simpler to reason about.

## First admin

After the first user signs up, promote them in the database:

```sql
UPDATE "User" SET role = 'ADMIN' WHERE email = 'you@example.com';
```

Gate settings/integration pages on `role === "ADMIN"`.

## Working with Claude Code

Build and verify in this order: credentials login (with a manually-seeded user) → signup + verification email → Google OAuth → Microsoft OAuth → middleware protection. Test each before moving on; auth bugs are easier to isolate one provider at a time.
