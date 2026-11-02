# تعلم NextAuth.js في Next.js من البداية حتى الاحتراف

_دليل عربي عملي شامل لـ Auth.js وNextAuth.js في Next.js، مع مصطلحات إنجليزية وأمثلة TypeScript — آخر مراجعة: 2 نوفمبر 2026_

---

## 📋 كيف تستخدم هذا الدليل

هذا الملف مرجع تعليمي تطبيقي يبدأ من المفاهيم الأساسية وينتهي بتصميم نظام مصادقة قابل للنشر في الإنتاج. الاسم التاريخي للمكتبة هو **NextAuth.js**، بينما تشير الوثائق الحديثة إلى تكامل Next.js باسم **Auth.js for Next.js** والحزمة `next-auth`. لذلك ستجد المصطلحين معًا، لكن الأمثلة الحديثة ستستخدم نمط `auth.ts` و`auth()` الخاص بالإصدار الخامس. [1]

> **ملاحظة مهمة عن حالة المشروع:** أصبح Auth.js جزءًا من Better Auth، وتوصي صفحة المستودع الرسمي بأن تبدأ المشاريع الجديدة بـ Better Auth إلا إذا احتجت تحديدًا إلى بعض الفجوات، وأبرزها الجلسات عديمة الحالة من دون قاعدة بيانات. هذا الدليل يشرح NextAuth.js/Auth.js كما هو مطلوب، ويضع قرار استخدام Better Auth في موضعه الصحيح بدل إخفائه. [2] [3]

### خريطة التعلّم

```mermaid
flowchart LR
    accTitle: Auth.js Learning Path
    accDescr: مسار تدريجي يبدأ من مفاهيم المصادقة ثم إعداد Auth.js وإضافة المزودين والجلسات والتفويض، وينتهي بالأمن والاختبار والنشر.

    basics([📋 الأساسيات]) --> setup[🔧 التثبيت والإعداد]
    setup --> providers[🔌 مزودو تسجيل الدخول]
    providers --> sessions[🔐 الجلسات والكوكيز]
    sessions --> authorization[🛡️ التفويض والصلاحيات]
    authorization --> production[🚀 الإنتاج والأمن]
    production --> mastery([✅ الاحتراف])

    classDef primary fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#1e3a5f
    classDef success fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d
    class basics,setup,providers,sessions,authorization primary
    class production,mastery success
```

### ما الذي ستبنيه

سنفترض تطبيق Next.js حديثًا باستخدام App Router وTypeScript. سنبني نظامًا يدعم تسجيل الدخول عبر GitHub أو Google، مع حماية الصفحات والـ Route Handlers، وجلسة JWT أو Database، وRole-Based Access Control، وصفحة دخول مخصصة، وتهيئة مناسبة للبيئات المختلفة.

---

## 🧭 المصطلحات التي يجب ألا تختلط

### Authentication وAuthorization وSession

**Authentication أو AuthN** تجيب عن سؤال: «من أنت؟». يتحقق النظام من هوية المستخدم باستخدام OAuth أو OIDC أو البريد السحري أو بيانات اعتماد أخرى.

**Authorization أو AuthZ** تجيب عن سؤال: «ماذا يسمح لك أن تفعل؟». قد يكون المستخدم مسجل الدخول، لكنه لا يملك صلاحية دخول لوحة الإدارة.

**Session management** تحفظ حالة تسجيل الدخول بين الطلبات. بعد نجاح AuthN، يضع النظام جلسة أو معرف جلسة في Cookie آمنة، ثم يقرأها في الطلبات اللاحقة. يميز Next.js رسميًا بين المفاهيم الثلاثة ويؤكد أن اختيار مكتبة مصادقة يقلل التعقيد والمخاطر. [4]

| المفهوم | السؤال | مثال في Next.js |
|---|---|---|
| Authentication | من هو المستخدم؟ | التحقق من رد Google أو كلمة المرور |
| Session | هل ما زال تسجيل الدخول فعالًا؟ | `const session = await auth()` |
| Authorization | هل يملك الصلاحية؟ | `session.user.role === "admin"` |

### OAuth وOIDC وCredentials وMagic Link

- **OAuth 2.0** إطار تفويض يسمح للتطبيق بطلب الوصول إلى موارد لدى مزود خارجي.
- **OpenID Connect أو OIDC** طبقة هوية مبنية فوق OAuth وتضيف هوية المستخدم و`id_token`.
- **Credentials** يسمح لك بربط Auth.js بنظام اسم مستخدم وكلمة مرور موجود. لا يعني أن Auth.js أصبح مسؤولًا تلقائيًا عن تخزين كلمات المرور أو استرجاعها.
- **Email/Magic Link** يرسل رابطًا قصير العمر إلى البريد. عند فتحه، يتحقق النظام من رمز التحقق.
- **Passkeys/WebAuthn** مصادقة باستخدام مفتاح الجهاز أو القياسات الحيوية. دعمها في Auth.js موصوف بأنه تجريبي وغير موصى به للإنتاج في الوثائق الحالية. [5]

---

## 🧱 بنية Auth.js في Next.js

في النمط الحديث، تضع إعدادًا واحدًا في `auth.ts` وتصدر منه دوالًا تستخدمها بقية أجزاء التطبيق. ثم يربط Route Handler هذه الدوال بالمسارين `GET` و`POST`.

```mermaid
sequenceDiagram
    accTitle: Auth.js Sign In Flow
    accDescr: يوضح انتقال المستخدم من صفحة الدخول إلى Auth.js ثم مزود الهوية، وعودة النتيجة إلى Route Handler ثم إنشاء الجلسة.

    participant user as 👤 المستخدم
    participant app as 🌐 Next.js
    participant auth as 🔐 Auth.js
    participant provider as 🔌 OAuth provider
    participant cookie as 💾 Browser cookie

    user->>app: يضغط Sign in
    app->>auth: signIn(provider)
    auth->>provider: Authorization request
    provider-->>user: صفحة الموافقة
    user->>provider: يوافق
    provider-->>auth: Callback code
    auth->>provider: Exchange code for tokens
    auth->>cookie: يضع جلسة HttpOnly
    auth-->>app: Redirect إلى التطبيق
    app-->>user: صفحة محمية
```

### الملفات الأساسية

```text
project-root/
├── auth.ts
├── proxy.ts                         # Next.js 16+؛ كان middleware.ts في الإصدارات الأقدم
├── app/
│   ├── api/auth/[...nextauth]/route.ts
│   ├── auth/signin/page.tsx
│   └── dashboard/page.tsx
├── types/
│   └── next-auth.d.ts
└── .env.local
```

وفق وثائق Auth.js الحالية، أصبح `proxy.ts` الاسم المستخدم في Next.js 16 بدل `middleware.ts`. إذا كنت على إصدار أقدم، استخدم `middleware.ts` بالطريقة نفسها تقريبًا. [6]

---

## 🧰 المتطلبات والتثبيت

### المتطلبات

| المتطلب | التوصية | التحقق |
|---|---|---|
| Node.js | إصدار يدعمه إصدار Next.js في مشروعك | `node --version` |
| Next.js | 14 أو أحدث لنمط Auth.js v5 | `npx next --version` |
| TypeScript | مفضل بشدة | `npx tsc --version` |
| OAuth app | GitHub أو Google أو مزود OIDC | من لوحة المزود |
| Secret | قيمة عشوائية عالية الإنتروبيا | `npx auth secret` |

أنشئ مشروعًا جديدًا إن لم يكن لديك مشروع:

```bash
npx create-next-app@latest next-auth-lab
cd next-auth-lab
npm install next-auth@beta
```

وثائق Auth.js الرسمية الحالية تستخدم `next-auth@beta` في تعليمات التثبيت. تحقّق من الإصدار الفعلي قبل تثبيت مشروع جديد، لأن وسم `beta` قد يتغير بمرور الوقت. [6]

أنشئ السر:

```bash
npx auth secret
```

أو:

```bash
openssl rand -base64 32
```

ثم احفظه في `.env.local`:

```dotenv
AUTH_SECRET="ضع_قيمة_عشوائية_طويلة_هنا"
```

لا تضع هذا الملف في Git. في الإنتاج، اجعل `AUTH_SECRET` سرًا حقيقيًا ثابتًا ولا تغيّره مع كل نشر، وإلا ستصبح الجلسات القديمة غير قابلة للفك.

### متغيرات البيئة الحديثة

يدعم Auth.js الاستدلال التلقائي لأسماء مزودي OAuth:

```dotenv
AUTH_SECRET="..."
AUTH_GITHUB_ID="..."
AUTH_GITHUB_SECRET="..."
AUTH_GOOGLE_ID="..."
AUTH_GOOGLE_SECRET="..."
```

الصيغة العامة هي `AUTH_{PROVIDER}_{ID|SECRET}`. ما زالت الأسماء القديمة `NEXTAUTH_SECRET` و`NEXTAUTH_URL` معروفة في سياقات التوافق، لكن يفضل استخدام بادئة `AUTH_` في الإعدادات الجديدة. [1]

> **قاعدة أمنية:** كل متغير يبدأ بـ `NEXT_PUBLIC_` أو `PUBLIC_` بحسب الإطار قد يصل إلى المتصفح. لا تضع Client Secret أو `AUTH_SECRET` في متغير عام.

---

## 🛠️ أول إعداد يعمل باستخدام GitHub

### ملف `auth.ts`

أنشئ الملف في جذر المشروع:

```ts
// auth.ts
import NextAuth from "next-auth"
import GitHub from "next-auth/providers/github"

export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [GitHub],
})
```

### Route Handler

```ts
// app/api/auth/[...nextauth]/route.ts
import { handlers } from "@/auth"

export const { GET, POST } = handlers
```

إذا لم يكن alias `@/*` مضبوطًا، استخدم مسارًا نسبيًا مناسبًا.

### صفحة محمية

```tsx
// app/dashboard/page.tsx
import { auth } from "@/auth"
import { redirect } from "next/navigation"

export default async function DashboardPage() {
  const session = await auth()

  if (!session) {
    redirect("/auth/signin")
  }

  return (
    <main>
      <h1>مرحبًا {session.user?.name ?? "بك"}</h1>
      <p>هذه الصفحة لا تظهر إلا للمستخدم المسجل دخوله.</p>
    </main>
  )
}
```

### تسجيل الدخول والخروج من Server Action

```ts
// app/actions/auth.ts
"use server"

import { signIn, signOut } from "@/auth"
import { AuthError } from "next-auth"

export async function signInWithGitHub() {
  await signIn("github", { redirectTo: "/dashboard" })
}

export async function signOutUser() {
  await signOut({ redirectTo: "/" })
}

export async function signInWithGitHubSafely() {
  try {
    await signIn("github", { redirectTo: "/dashboard" })
  } catch (error) {
    if (error instanceof AuthError) {
      return { error: "تعذر تسجيل الدخول" }
    }
    throw error
  }
}
```

لا تبتلع أخطاء `redirect` العامة داخل `try/catch` بلا تمييز؛ فالدوال التي تنفذ Redirect قد تستخدم آلية استثناء داخلية في Next.js. تعامل مع `AuthError` المعروف، ثم أعد رمي الأخطاء الأخرى.

### زر عميل بسيط

```tsx
// app/components/auth-buttons.tsx
"use client"

import { signIn, signOut } from "next-auth/react"

export function SignInButton() {
  return (
    <button type="button" onClick={() => signIn("github", { redirectTo: "/dashboard" })}>
      تسجيل الدخول عبر GitHub
    </button>
  )
}

export function SignOutButton() {
  return (
    <button type="button" onClick={() => signOut({ redirectTo: "/" })}>
      تسجيل الخروج
    </button>
  )
}
```

في App Router، يفضل استخدام `auth()` في Server Components. استخدم `next-auth/react` و`SessionProvider` عندما تحتاج حالة جلسة تفاعلية في Client Components فقط. توضح وثائق React أن `signIn` و`signOut` و`useSession` موجهة للعميل، بينما دوال Server Actions تصدر من إعداد `auth.ts`. [7]

### Proxy أو Middleware للحماية المبكرة

```ts
// proxy.ts — Next.js 16+
export { auth as proxy } from "@/auth"
```

هذا مفيد للتحقق الأولي وإعادة التوجيه، لكنه ليس بديلًا عن فحص الصلاحية داخل الصفحة أو Route Handler الذي يقرأ البيانات. لا تعتمد على إخفاء رابط في واجهة المستخدم كآلية حماية.

---

## 🔌 مزودو تسجيل الدخول

### OAuth provider مدمج

غالبًا يكفي استيراد المزود وإضافته إلى `providers`:

```ts
import NextAuth from "next-auth"
import Google from "next-auth/providers/google"
import GitHub from "next-auth/providers/github"

export const { handlers, auth } = NextAuth({
  providers: [Google, GitHub],
})
```

يضم Auth.js عددًا كبيرًا من مزودي OAuth المهيئين مسبقًا، مع دعم OAuth 2.0 وOIDC ومزودين معروفين مثل Google وGitHub وKeycloak وOkta. [8]

### تخصيص مزود OAuth

إذا احتجت Scope أو `profile` مختلفًا، مرر إعدادًا إلى دالة المزود:

```ts
import Auth0 from "next-auth/providers/auth0"

Auth0({
  authorization: {
    params: {
      scope: "openid profile email custom_scope",
    },
  },
  async profile(profile) {
    return {
      id: profile.sub,
      name: profile.name,
      email: profile.email,
      image: profile.picture,
      department: profile.department,
    }
  },
})
```

يتم دمج الإعدادات المخصصة مع الإعدادات الافتراضية للمزود، لذلك لا تحتاج عادة إلى إعادة كتابة كائن `authorization` كامل. [9]

### مزود OIDC مخصص

```ts
import NextAuth from "next-auth"

export const { handlers, auth } = NextAuth({
  providers: [
    {
      id: "company-sso",
      name: "Company SSO",
      type: "oidc",
      issuer: process.env.AUTH_COMPANY_ISSUER,
      clientId: process.env.AUTH_COMPANY_ID,
      clientSecret: process.env.AUTH_COMPANY_SECRET,
    },
  ],
})
```

يكون Callback URL الافتراضي في Next.js بالشكل:

```text
https://example.com/api/auth/callback/company-sso
```

إذا غيّرت `basePath`، يجب أن تسجل المسار الناتج في لوحة مزود الهوية. [9]

### Credentials provider

يستخدم Credentials عندما تكون لديك خدمة مستخدمين موجودة أو تريد تنفيذ منطق تحقق خاص:

```ts
import Credentials from "next-auth/providers/credentials"
import NextAuth, { CredentialsSignin } from "next-auth"
import { z } from "zod"

class InvalidLoginError extends CredentialsSignin {
  code = "Invalid identifier or password"
}

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    Credentials({
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      async authorize(credentials) {
        const parsed = z
          .object({ email: z.string().email(), password: z.string().min(8) })
          .safeParse(credentials)

        if (!parsed.success) return null

        const response = await fetch(`${process.env.AUTH_API_URL}/login`, {
          method: "POST",
          headers: { "content-type": "application/json" },
          body: JSON.stringify(parsed.data),
          cache: "no-store",
        })

        if (!response.ok) throw new InvalidLoginError()

        const user = await response.json()
        if (!user?.id || !user.email) return null

        return {
          id: String(user.id),
          name: user.name ?? null,
          email: user.email,
          role: user.role ?? "user",
        }
      },
    }),
  ],
})
```

ينبغي ألا تخزن كلمات المرور بنصها الصريح. استخدم خدمة هوية متخصصة أو خزّن Hash قويًا باستخدام Argon2id أو bcrypt وفق متطلبات مشروعك، وأضف rate limiting وMFA واسترجاع كلمة مرور آمنًا. وثائق Auth.js تحذر من أن Credentials لا توفر تلقائيًا منظومة password management وabuse detection التي يوفرها مزود هوية ناضج. [10]

لا تضع تفاصيل مثل «البريد صحيح لكن كلمة المرور خطأ» في رسالة ظاهرة للمهاجم. يمكن أن يظهر `code` في URL، ولذلك يجب أن يكون عامًا ولا يكشف معلومة حساسة. [10]

### Magic Links عبر البريد

يحتاج Email provider عادةً إلى Adapter لتخزين رموز التحقق والمستخدمين. مثال مفاهيمي باستخدام Resend:

```ts
import NextAuth from "next-auth"
import Resend from "next-auth/providers/resend"
import { PrismaAdapter } from "@auth/prisma-adapter"
import { prisma } from "@/lib/prisma"

export const { handlers, auth } = NextAuth({
  adapter: PrismaAdapter(prisma),
  providers: [
    Resend({
      from: "auth@example.com",
    }),
  ],
})
```

يجب ضبط `AUTH_RESEND_KEY` أو المتغيرات الخاصة بالمزود وفق صفحة المزود المختار، وتهيئة Domain موثق في خدمة البريد.

### Passkeys وWebAuthn

مزود Passkey يحتاج Adapter وجدولًا إضافيًا، وتصف الوثائق الحالية دعمه بأنه **experimental**. لا تفعله في نظام إنتاج حساس قبل مراجعة حالة الإصدار والاختبارات الأمنية الخاصة بالمشروع. [11]

---

## 🗃️ الجلسات: JWT أم Database؟

يدعم Auth.js استراتيجيتين رئيسيتين. إذا لم تستخدم Adapter، يكون JWT هو الافتراضي عادةً. وإذا استخدمت Adapter، تصبح Database هي الافتراضية عادةً، ويمكنك فرض JWT صراحةً. [12]

### JWT session

```ts
export const { handlers, auth } = NextAuth({
  session: { strategy: "jwt" },
  providers: [GitHub],
})
```

في هذه الاستراتيجية، تحفظ جلسة مشفرة في Cookie `HttpOnly`. ميزتها أنها لا تحتاج إلى قراءة قاعدة بيانات لكل تحقق، لكن إلغاء Token قبل انتهاء صلاحيته أصعب، وحجم Cookie محدود عمليًا بنحو 4KB لكل Cookie. يطبق Auth.js Cookie chunking عندما تتجاوز القيمة الحد، لكن البيانات تنتقل مع الطلبات، لذلك لا تضع بيانات كثيرة في JWT. [12]

### Database session

```ts
export const { handlers, auth } = NextAuth({
  adapter: PrismaAdapter(prisma),
  session: {
    strategy: "database",
    maxAge: 30 * 24 * 60 * 60,
    updateAge: 24 * 60 * 60,
  },
  providers: [GitHub],
})
```

تحفظ Cookie معرف جلسة فقط، بينما تبقى بيانات الجلسة في قاعدة البيانات. يسهل ذلك «تسجيل الخروج من كل الأجهزة» وإبطال الجلسات مركزيًا، لكنه يضيف قراءة قاعدة بيانات وتحديات اتصال في Edge. [12]

| المعيار | JWT | Database |
|---|---|---|
| قاعدة بيانات للجلسة | لا | نعم |
| الإبطال الفوري | أصعب | مباشر |
| الأداء في التحقق | غالبًا بلا DB round trip | يحتاج DB أو caching |
| Edge compatibility | أسهل | يعتمد على Adapter/ORM |
| حجم البيانات | محدود بحجم Cookie | أكبر، لأن البيانات خارج Cookie |
| مناسب لـ | تطبيقات stateless وخدمات بسيطة | التحكم المركزي والجلسات القابلة للإبطال |

### ما الذي يجب تخزينه في الجلسة؟

خزن أقل قدر ممكن: `id` و`role` أو Claim يحتاجه التطبيق. لا تضع Access Token أو Refresh Token في Session object المرسل إلى العميل إلا إذا كان ذلك مقصودًا ومدروسًا. الافتراضي يعرض `name` و`email` و`image` فقط حتى لا تُكشف بيانات أخرى بالخطأ. [13]

---

## 🗄️ Adapter مع Prisma

### التثبيت

```bash
npm install @auth/prisma-adapter @prisma/client
npm install -D prisma
npx prisma init
```

ضع الاتصال:

```dotenv
DATABASE_URL="postgresql://user:password@host:5432/app"
```

### Prisma singleton

```ts
// lib/prisma.ts
import { PrismaClient } from "@prisma/client"

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma =
  globalForPrisma.prisma ?? new PrismaClient()

if (process.env.NODE_ENV !== "production") {
  globalForPrisma.prisma = prisma
}
```

### Schema قياسي

```prisma
// prisma/schema.prisma
model User {
  id            String    @id @default(cuid())
  name          String?
  email         String?   @unique
  emailVerified DateTime?
  image         String?
  role          String    @default("user")
  accounts      Account[]
  sessions      Session[]
}

model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?
  user              User    @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}
```

نفذ:

```bash
npx prisma migrate dev --name add_auth
npx prisma generate
```

### ربط Adapter

```ts
// auth.ts
import NextAuth from "next-auth"
import GitHub from "next-auth/providers/github"
import { PrismaAdapter } from "@auth/prisma-adapter"
import { prisma } from "@/lib/prisma"

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(prisma),
  providers: [GitHub],
  session: { strategy: "database" },
})
```

توصي وثائق Prisma Adapter باستخدام الحزمة الحديثة `@auth/prisma-adapter` بدل الاسم القديم `@next-auth/prisma-adapter`. راجع أيضًا توافق Prisma مع Edge ونسخة Next.js المستخدمة لديك. [14]

### ما الذي ينشئه Adapter؟

ينفذ Adapter عمليات مثل:

- إنشاء واسترجاع وتحديث وحذف `User`.
- ربط حساب OAuth في `Account` بالمستخدم.
- إنشاء والتحقق وحذف `Session`.
- إنشاء واستهلاك `VerificationToken` للبريد.

يمكنك كتابة Adapter خاص بك، لكنه عقد API حساس؛ اختبر كل طريقة، والتعامل مع حالات السباق، والمفاتيح الفريدة، والتواريخ، وحذف السجلات التابعة. [15]

---

## 🧩 قراءة الجلسة في كل سياق

### Server Component

```tsx
import { auth } from "@/auth"

export default async function Page() {
  const session = await auth()
  return <pre>{JSON.stringify(session, null, 2)}</pre>
}
```

### Route Handler

```ts
// app/api/me/route.ts
import { auth } from "@/auth"
import { NextResponse } from "next/server"

export async function GET() {
  const session = await auth()

  if (!session) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
  }

  return NextResponse.json({ user: session.user })
}
```

### Server Action

```ts
"use server"

import { auth } from "@/auth"
import { redirect } from "next/navigation"

export async function updateProfile(formData: FormData) {
  const session = await auth()
  if (!session) redirect("/auth/signin")

  // تحقق من المدخلات والصلاحية قبل mutation.
}
```

### Client Component مع `useSession`

```tsx
"use client"

import { useSession } from "next-auth/react"

export function UserMenu() {
  const { data: session, status } = useSession({ required: false })

  if (status === "loading") return <p>جارٍ التحقق...</p>
  if (!session) return <p>زائر</p>

  return <p>{session.user?.email}</p>
}
```

لتمكين `useSession`، أضف Provider في Client Component:

```tsx
// app/providers.tsx
"use client"

import { SessionProvider } from "next-auth/react"

export function Providers({ children }: { children: React.ReactNode }) {
  return <SessionProvider>{children}</SessionProvider>
}
```

```tsx
// app/layout.tsx
import { Providers } from "./providers"

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="ar" dir="rtl">
      <body><Providers>{children}</Providers></body>
    </html>
  )
}
```

في App Router لا تضف `SessionProvider` للتطبيق كله بلا حاجة؛ `auth()` أبسط وأقل Client JavaScript في كثير من الصفحات. استخدم Provider عندما تحتاج `useSession`, tab synchronization, أو تحديث الجلسة من العميل. [7]

### Pages Router legacy

إذا كان مشروعك يستخدم `pages/`، يبقى النمط القديم مدعومًا:

```ts
// pages/api/auth/[...nextauth].ts — v4-style
import NextAuth from "next-auth"
import GitHub from "next-auth/providers/github"

export default NextAuth({
  providers: [GitHub],
})
```

وفي الصفحات:

```tsx
import { getServerSession } from "next-auth/next"
import { authOptions } from "./api/auth/[...nextauth]"

export async function getServerSideProps(context: any) {
  const session = await getServerSession(context.req, context.res, authOptions)
  return { props: { session } }
}
```

في v5 يفضل الانتقال إلى `auth()` بدل `getServerSession` عندما تسمح بنية المشروع بذلك. دليل الترحيل الرسمي يصف v5 بأنه App Router-first مع استمرار دعم Pages Router. [16]

---

## 🛡️ التفويض وحماية الموارد

### حماية صفحة كاملة

```tsx
import { auth } from "@/auth"
import { redirect } from "next/navigation"

export default async function AdminPage() {
  const session = await auth()

  if (!session) redirect("/auth/signin")
  if (session.user.role !== "admin") redirect("/forbidden")

  return <h1>لوحة الإدارة</h1>
}
```

### حماية البيانات لا المسار فقط

```ts
export async function DELETE() {
  const session = await auth()
  if (!session) return new Response("Unauthorized", { status: 401 })
  if (session.user.role !== "admin") {
    return new Response("Forbidden", { status: 403 })
  }

  // تحقق أيضًا من ملكية السجل قبل حذفه.
  return new Response(null, { status: 204 })
}
```

لا يكفي وضع الفحص في Layout أو Proxy فقط؛ يجب أن يفحص كل endpoint العملية والملكية والصلاحية المطلوبة. حتى إرشادات الترحيل إلى Better Auth تحذر من استخدام Proxy كحل كامل لجلب البيانات والتفويض. [3]

### RBAC مع JWT

```ts
callbacks: {
  jwt({ token, user }) {
    if (user) token.role = user.role
    return token
  },
  session({ session, token }) {
    session.user.role = token.role as string
    return session
  },
}
```

في JWT، إذا تغيرت رتبة المستخدم في قاعدة البيانات، قد تحتاج إلى إعادة إصدار الجلسة أو تسجيل دخوله من جديد، لأن القيمة موجودة في Token. في Database session، يمكنك قراءة الدور الحالي من User مباشرة. [17]

### السماح بنطاق بريد معين

```ts
callbacks: {
  async signIn({ profile }) {
    const email = profile?.email
    return typeof email === "string" && email.endsWith("@example.com")
  },
}
```

لا تعتبر Domain وحده دليلًا كافيًا في كل سياق أمني؛ فعّل التحقق من البريد وسياسات المؤسسة ومصدر الهوية وفق مستوى الخطر. [18]

---

## 🔁 Callbacks وEvents

### `signIn`

يقرر هل يسمح النظام بتسجيل الدخول. أعد `true` للسماح و`false` للرفض، أو أعد مسارًا إذا كان السيناريو يتطلب تحويلًا خاصًا بحسب إصدار Auth.js.

```ts
callbacks: {
  async signIn({ user, account, profile }) {
    if (account?.provider === "github" && !profile?.email) {
      return "/auth/error?reason=email_required"
    }
    return Boolean(user.email)
  },
}
```

### `redirect`

تحكم في وجهة التحويل. لا تسمح بـ open redirect:

```ts
callbacks: {
  async redirect({ url, baseUrl }) {
    if (url.startsWith("/")) return `${baseUrl}${url}`
    if (new URL(url).origin === baseUrl) return url
    return baseUrl
  },
}
```

### `jwt`

يُستدعى عند إنشاء JWT أو تحديثه. في أول تسجيل دخول تتوفر عادةً معلومات `user` و`account` و`profile`؛ في الطلبات اللاحقة تعتمد على `token`. لا تحفظ أي Claim لا يحتاجه التطبيق.

```ts
callbacks: {
  async jwt({ token, user, account }) {
    if (user) token.userId = user.id
    if (account) token.provider = account.provider
    return token
  },
}
```

### `session`

يحدد ما يخرج إلى Session object. هذا ليس مكانًا لإرسال الأسرار إلى العميل.

```ts
callbacks: {
  async session({ session, token }) {
    session.user.id = token.userId as string
    return session
  },
}
```

### `authorized`

يستخدم في إعدادات الحماية التي تستعمل `auth` كـ Proxy wrapper:

```ts
callbacks: {
  authorized({ auth, request }) {
    const isLoggedIn = Boolean(auth?.user)
    const isPublic = request.nextUrl.pathname.startsWith("/public")
    return isPublic || isLoggedIn
  },
}
```

### Events

الأحداث مناسبة للتدقيق Audit Logging لأنها لا تغير قرار الطلب عادةً:

```ts
events: {
  async signIn(message) {
    console.info("auth.signIn", {
      userId: message.user.id,
      provider: message.account?.provider,
    })
  },
  async signOut(message) {
    console.info("auth.signOut", message)
  },
}
```

لا تطبع Access Tokens أو Refresh Tokens أو كلمات المرور في السجلات. استخدم `logger` المخصص وأخفِ البريد أو عناوين IP عندما تكون سياسة الخصوصية تتطلب ذلك.

---

## 🧬 TypeScript وModule Augmentation

افتراضيًا لا يحتوي `session.user` على `id` و`role` في النوع القياسي. أضف ملفًا مثل `types/next-auth.d.ts`:

```ts
import "next-auth"
import "next-auth/jwt"
import type { DefaultSession } from "next-auth"

declare module "next-auth" {
  interface User {
    role?: "user" | "admin"
  }

  interface Session {
    user: {
      id: string
      role?: "user" | "admin"
    } & DefaultSession["user"]
    error?: "RefreshTokenError"
  }
}

declare module "next-auth/jwt" {
  interface JWT {
    userId?: string
    role?: "user" | "admin"
    provider?: string
    error?: "RefreshTokenError"
  }
}
```

تأكد أن `tsconfig.json` يشمل المجلد:

```json
{
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", "types/**/*.d.ts"]
}
```

ثم افحص:

```bash
npx tsc --noEmit
```

لا تستخدم `as any` لإخفاء مشكلة في شكل الجلسة. عدّل النوع والـ callbacks معًا، وابقِ البيانات التي تكشفها صريحة ومحدودة.

---

## 🔄 Refresh Token Rotation

لا يوفر Auth.js حتى الآن حلًا مدمجًا عديم الإعداد للتدوير التلقائي لـ Refresh Token. توضح الوثائق الرسمية نمطًا يعتمد على `jwt` و`session` callbacks. [19]

```ts
callbacks: {
  async jwt({ token, account }) {
    if (account) {
      return {
        ...token,
        access_token: account.access_token,
        expires_at: account.expires_at,
        refresh_token: account.refresh_token,
      }
    }

    if (!token.expires_at || Date.now() < token.expires_at * 1000) {
      return token
    }

    if (!token.refresh_token) {
      token.error = "RefreshTokenError"
      return token
    }

    try {
      const response = await fetch("https://provider.example.com/oauth/token", {
        method: "POST",
        headers: { "content-type": "application/x-www-form-urlencoded" },
        body: new URLSearchParams({
          client_id: process.env.AUTH_PROVIDER_ID!,
          client_secret: process.env.AUTH_PROVIDER_SECRET!,
          grant_type: "refresh_token",
          refresh_token: token.refresh_token,
        }),
        cache: "no-store",
      })

      if (!response.ok) throw new Error("Refresh failed")
      const refreshed = await response.json()

      return {
        ...token,
        access_token: refreshed.access_token,
        expires_at: Math.floor(Date.now() / 1000 + refreshed.expires_in),
        refresh_token: refreshed.refresh_token ?? token.refresh_token,
      }
    } catch {
      return { ...token, error: "RefreshTokenError" }
    }
  },
  async session({ session, token }) {
    session.error = token.error
    return session
  },
}
```

انتبه إلى Race Condition: بعض المزودين يبطلون Refresh Token بعد استخدامه مرة واحدة. قد تطلب عدة تبويبات التحديث في اللحظة نفسها. استخدم قفلًا أو cache موزعًا أو تصميمًا يقلل التحديث المتزامن، ولا تفترض أن المثال العام يحل كل ظروف الإنتاج. [19]

---

## 🌐 Edge وProxy وORM

يستخدم Auth.js Web APIs قياسية، لكن مكتبة قاعدة البيانات أو ORM قد تعتمد على Node APIs أو TCP sockets غير المتاحة في Edge. لذلك قد يفشل استيراد Adapter داخل Proxy أو Middleware القديم. [20]

### نمط فصل الإعداد

```ts
// auth.config.ts
import GitHub from "next-auth/providers/github"
import type { NextAuthConfig } from "next-auth"

export default {
  providers: [GitHub],
} satisfies NextAuthConfig
```

```ts
// auth.ts — الإعداد الكامل في Node
import NextAuth from "next-auth"
import { PrismaAdapter } from "@auth/prisma-adapter"
import { prisma } from "@/lib/prisma"
import authConfig from "@/auth.config"

export const { auth, handlers, signIn, signOut } = NextAuth({
  ...authConfig,
  adapter: PrismaAdapter(prisma),
  session: { strategy: "jwt" },
})
```

```ts
// proxy.ts — يقرأ الإعداد المتوافق فقط
import NextAuth from "next-auth"
import authConfig from "@/auth.config"

export const { auth: proxy } = NextAuth(authConfig)
```

في Next.js 16، قد يعمل `proxy.ts` على Node runtime بحسب إعدادات الإصدار، لكن لا تفترض ذلك دون قراءة وثائق نسخة مشروعك. أبقِ الاستعلامات الثقيلة والصلاحيات النهائية داخل الصفحة أو Route Handler. [20]

---

## 🎨 صفحات الدخول المخصصة والإتاحة

### إعداد المسارات

```ts
export const { handlers, auth } = NextAuth({
  pages: {
    signIn: "/auth/signin",
    error: "/auth/error",
  },
  providers: [GitHub],
})
```

### صفحة دخول قابلة للوصول

```tsx
// app/auth/signin/page.tsx
import { signInWithGitHub } from "@/app/actions/auth"

export default function SignInPage() {
  return (
    <main>
      <h1>تسجيل الدخول</h1>
      <p id="signin-help">اختر طريقة آمنة للمتابعة إلى التطبيق.</p>

      <form action={signInWithGitHub} aria-describedby="signin-help">
        <button type="submit">المتابعة عبر GitHub</button>
      </form>
    </main>
  )
}
```

قائمة إتاحة عملية:

- استخدم `label` مرتبطًا بـ`input` عبر `htmlFor` و`id`.
- لا تعتمد على اللون وحده لعرض الخطأ أو حالة التركيز.
- اجعل زر التسجيل عنصر `<button>` حقيقيًا، وليس `<div onClick>`.
- اعرض رسالة خطأ مفهومة قرب الحقل، واربطها بـ`aria-describedby`.
- حافظ على ترتيب Tab منطقي ومؤشر Focus مرئي.
- اجعل الهدف التفاعلي مناسبًا للمس، ويفضل ألا يقل عن 44×44 CSS pixels.
- لا تعرض رسالة «البريد موجود» أو «المستخدم غير موجود» إذا كانت تكشف وجود حساب.
- أضف `lang="ar"` و`dir="rtl"` للتطبيق العربي، مع إبقاء البريد والرموز في اتجاه مناسب عند الحاجة.

تغطي هذه النقاط مبادئ WCAG 2.1 AA المتعلقة بالبديل النصي، البنية الدلالية، التباين، لوحة المفاتيح، التركيز، واسم/دور/قيمة عناصر الواجهة. [21]

---

## 🔒 نموذج الأمن في Auth.js

يهدف Auth.js إلى إعدادات آمنة افتراضيًا: POST routes محمية ضد CSRF، Cookies بإعدادات مقيدة، وJWT مشفر افتراضيًا عند استخدامه كجلسة. لكن المكتبة لا تمنع أخطاء تكامل التطبيق. [2]

### قائمة حماية الإنتاج

- أنشئ `AUTH_SECRET` عالي الإنتروبيا وخزنه في Secret Manager.
- استخدم HTTPS في تسجيل الدخول وكل الصفحات المحمية.
- لا تسجل Tokens أو كلمات المرور أو رموز التحقق.
- لا ترسل `access_token` و`refresh_token` إلى Client Session بلا حاجة واضحة.
- قيد `redirectTo` و`redirect` لمنع Open Redirect.
- تحقق من الصلاحية داخل كل mutation وRoute Handler.
- أضف rate limiting لمحاولات Credentials وMagic Link.
- استخدم رسائل دخول عامة لتقليل User Enumeration.
- فعّل MFA أو Passkeys عبر مزود هوية مناسب للحسابات الحساسة.
- راجع Scopes ولا تطلب صلاحيات OAuth لا تحتاجها.
- راجع Callback URLs في لوحة كل مزود، وفصل Development عن Production.
- اكتب Audit Logs للأحداث المهمة دون كشف أسرار.
- اختبر إبطال الجلسة، انتهاءها، وتسجيل الخروج من كل الأجهزة.
- راجع التبعيات دوريًا، وثبت الإصدارات في تطبيق إنتاج.

توصي OWASP باستخدام TLS، تقليل كشف معلومات الأخطاء، إعادة المصادقة للعمليات الحساسة، وحماية كلمات المرور من التخمين وإعادة الاستخدام. [22]

### Cookies وCSRF وXSS

Cookie `HttpOnly` تمنع JavaScript العادي من قراءتها، لكنها لا تجعل التطبيق محصنًا من XSS؛ فإذا تمكن مهاجم من تنفيذ JavaScript في الصفحة فقد يرسل طلبات نيابة عن المستخدم. لذلك اجمع بين CSP، تعقيم المخرجات، تحقق المدخلات، وCSRF protection.

لا تعطّل إعدادات Cookie الافتراضية أو تغير `secure`, `sameSite`, `httpOnly` إلا إذا فهمت أثر ذلك على كل بيئة. أي Cookie مخصص يجب أن يطابق اسمه وإعداداته في كل موضع يقرأه Auth.js.

---

## 🧪 الاختبار والتحقق

### فحوصات ثابتة

```bash
npm run lint
npx tsc --noEmit
npm run build
```

### سيناريوهات E2E أساسية

اختبر على الأقل:

1. زائر يفتح صفحة عامة.
2. زائر يفتح صفحة محمية ويعاد إلى تسجيل الدخول.
3. نجاح OAuth وعودة Redirect صحيحة.
4. رفض المستخدم من `signIn` callback.
5. انتهاء الجلسة وعودة `auth()` بقيمة `null`.
6. تسجيل الخروج ثم فشل الوصول إلى البيانات المحمية.
7. مستخدم عادي يحاول الوصول إلى Admin endpoint ويحصل على 403.
8. Redirect خارجي مرفوض.
9. Cookies تحمل `HttpOnly` و`Secure` في الإنتاج.
10. فشل مزود البريد أو OAuth يظهر برسالة عامة ولا يكشف Secret.
11. تجديد Access Token مع Refresh Token منتهٍ أو مفقود.
12. فتح تبويبين وتزامن تسجيل الخروج.

### اختبار وحدة لـ Authorization

```ts
export function canManageBilling(role: string | undefined) {
  return role === "admin" || role === "billing"
}
```

```ts
import { describe, expect, it } from "vitest"
import { canManageBilling } from "./permissions"

describe("canManageBilling", () => {
  it("allows admins", () => {
    expect(canManageBilling("admin")).toBe(true)
  })

  it("rejects normal users", () => {
    expect(canManageBilling("user")).toBe(false)
  })
})
```

اختبر الدالة الأمنية بمعزل عن الواجهة، ثم اختبر Route Handler فعلًا للتأكد من أن التحقق استُدعي قبل قاعدة البيانات أو mutation.

---

## 🐞 استكشاف الأخطاء

### `MissingSecretError`

**السبب:** لا توجد `AUTH_SECRET` أو `NEXTAUTH_SECRET` في البيئة التي يعمل فيها الخادم.

**الإصلاح:**

```bash
npx auth secret
```

ثم أعد تشغيل خادم التطوير وتأكد أن Secret موجود في إعدادات الإنتاج، لا في جهازك فقط.

### `CallbackRouteError` أو فشل OAuth

افحص:

- Client ID وClient Secret.
- Callback URL المسجل حرفيًا.
- `AUTH_*` باسم المزود الصحيح.
- اختلاف `http` و`https` أو المنفذ.
- أن النظام لا يزيل Query Parameters المطلوبة.
- وقت الجهاز وشهادة TLS.

### `redirect_uri_mismatch`

اجعل القيمة في لوحة المزود مساوية للمسار الفعلي:

```text
http://localhost:3000/api/auth/callback/google
https://example.com/api/auth/callback/google
```

لا تستخدم Callback URL إنتاجية في بيئة محلية.

### `CredentialsSignin`

**السبب المعتاد:** `authorize` أعاد `null` أو رمى `CredentialsSignin`.

**الإصلاح:** تحقق من شكل المدخلات، سجّل سببًا داخليًا آمنًا على الخادم، وأظهر للمستخدم رسالة عامة مثل «بيانات الدخول غير صحيحة».

### الجلسة `null` رغم نجاح الدخول

تحقق من:

- وجود Route Handler في `app/api/auth/[...nextauth]/route.ts`.
- أن `AUTH_SECRET` ثابت بين الطلبات.
- أن Domain وSecure Cookie مناسبان للبيئة.
- عدم استخدام `useSession` بلا `SessionProvider`.
- أن الاستيراد يأتي من `@/auth` في الخادم، ومن `next-auth/react` في العميل.
- أن Proxy أو Cache لا يخزن صفحة مستخدم لجلسة مستخدم آخر.

### `JWTSessionError`

يظهر غالبًا عند تغيير Secret، أو تلف Cookie، أو اختلاف إعداد `encode/decode`. سجّل المستخدم خروجًا واحذف Cookie في بيئة التطوير، ثم تأكد أن Secret لا يتغير مع كل تشغيل.

### مشكلة Edge أو `Node.js module not supported`

لا تستورد ORM غير متوافق داخل Proxy أو Middleware القديم. طبّق نمط `auth.config.ts` و`auth.ts` المنفصل، أو انقل التنفيذ إلى Node runtime بحسب دعم إصدار Next.js المستعمل. [20]

### Login يعمل لكن Role قديم

في JWT، الدور محفوظ في Token حتى يعاد إصدار الجلسة. حدّثه عبر Server Action موثوق أو أجبر إعادة المصادقة. في Database session، اقرأ الدور من مصدره الحالي بدل نسخة قديمة في Session object.

---

## 🔁 الترحيل من NextAuth v4 إلى v5/Auth.js

### التغييرات الرئيسية

| v4 | v5/Auth.js |
|---|---|
| `pages/api/auth/[...nextauth].ts` يحتوي الإعداد كاملًا | `auth.ts` مركزي يصدر helpers |
| `getServerSession(authOptions)` | `auth()` |
| `getToken` في مواضع كثيرة | `auth()` غالبًا |
| `withAuth` | `auth`/Proxy wrapper |
| `@next-auth/*-adapter` | `@auth/*-adapter` |
| `middleware.ts` في الإصدارات القديمة | `proxy.ts` في Next.js 16 |
| `NEXTAUTH_*` شائع | يفضل `AUTH_*` مع inference |

دليل الترحيل الرسمي يذكر أن v5 أدخل إعدادًا موحدًا وUniversal `auth()`، مع استمرار دعم Pages Router، وغيّر بعض تفاصيل OAuth بسبب تشدد التوافق مع المواصفات. [16]

### الترحيل العملي

1. أنشئ `auth.ts` وانقل `providers` و`callbacks` و`adapter`.
2. صدّر `auth`, `handlers`, `signIn`, و`signOut` حسب الحاجة.
3. اجعل Route Handler يصدر `GET` و`POST` من `handlers`.
4. استبدل `getServerSession` بـ`auth()` تدريجيًا.
5. استبدل حزم Adapter القديمة بحزم `@auth/*-adapter`.
6. راجع `callbackUrl` واستعمل `redirectTo` في API الحديثة عند توفره.
7. اختبر OAuth Callback URLs وEmail وDatabase migrations.
8. لا تحذف v4 قبل نجاح الاختبارات في بيئة staging.

### لا تخلط أمثلة v4 وv5

إذا كان مشروعك مثبتًا على `next-auth@4`، استخدم وثائق v4 المتوافقة. إذا كنت تستخدم `auth.ts` و`auth()`, لا تنسخ بلا تعديل مثالًا يعتمد على `authOptions` و`getServerSession`.

---

## 🏗️ بنية مشروع احترافية مقترحة

```text
src/
├── auth.ts
├── auth.config.ts
├── proxy.ts
├── app/
│   ├── api/
│   │   └── auth/[...nextauth]/route.ts
│   ├── actions/
│   │   └── auth.ts
│   ├── auth/
│   │   ├── signin/page.tsx
│   │   └── error/page.tsx
│   ├── dashboard/page.tsx
│   └── admin/page.tsx
├── components/
│   ├── auth-buttons.tsx
│   └── user-menu.tsx
├── lib/
│   ├── prisma.ts
│   ├── permissions.ts
│   └── validation.ts
└── types/
    └── next-auth.d.ts
```

قسّم المسؤوليات:

- `auth.ts`: تكوين Auth.js الكامل ومصادر الخادم.
- `auth.config.ts`: الخيارات المتوافقة مع Proxy إذا احتجت Edge split.
- `permissions.ts`: دوال صلاحيات قابلة للاختبار، وليست شروطًا مبعثرة في JSX.
- `validation.ts`: مخططات Zod أو Valibot للمدخلات.
- `actions/auth.ts`: Server Actions التي تتعامل مع Redirect و`AuthError`.
- `components`: واجهة العميل فقط.

---

## 📚 مرجع سريع للدوال والخيارات

### الدوال

| الدالة | السياق | الاستخدام |
|---|---|---|
| `auth()` | Server Component/Route/Action | قراءة الجلسة |
| `signIn()` من `@/auth` | Server Action/server | بدء دخول مع Redirect |
| `signOut()` من `@/auth` | Server Action/server | إنهاء الجلسة |
| `handlers` | Route Handler | تصدير `GET` و`POST` |
| `signIn()` من `next-auth/react` | Client Component | دخول من زر عميل |
| `signOut()` من `next-auth/react` | Client Component | خروج من زر عميل |
| `useSession()` | Client Component | قراءة وتحديث الجلسة |
| `SessionProvider` | Client tree | توفير Context |
| `getProviders()` | Client/Pages Router | عرض المزودين |
| `getCsrfToken()` | Client/legacy | قراءة CSRF token عند الحاجة الخاصة |

### أهم الخيارات

| الخيار | الغرض |
|---|---|
| `providers` | قائمة OAuth/OIDC/Credentials/Email/Passkey |
| `adapter` | ربط قاعدة البيانات |
| `session.strategy` | `jwt` أو `database` |
| `session.maxAge` | مدة الجلسة بالثواني |
| `session.updateAge` | تكرار تحديث Database session |
| `pages` | مسارات `signIn`, `error`, وغيرها |
| `callbacks` | التحكم في الدخول والتحويل والجلسة والـ JWT |
| `events` | Audit events غير المعيدة عادةً |
| `cookies` | إعداد متقدم؛ لا تغيره بلا سبب |
| `debug` | سجلات تشخيصية؛ لا تفعله بلا مراقبة الأسرار |
| `logger` | ربط السجلات بخدمة Observability |
| `basePath` | تغيير مسار Auth API |
| `experimental.enableWebAuthn` | تفعيل Passkeys التجريبية |

---

## 🧠 قرارات التصميم في الإنتاج

### متى تختار JWT؟

اختر JWT عندما تحتاج جلسة stateless، أو لا تريد DB round trip لكل تحقق، أو ستنشر Proxy/Edge مع مزود لا يدعم Database session. قلل البيانات، وقصر مدة الجلسة، وفكر في استراتيجية إبطال عند المخاطر العالية.

### متى تختار Database session؟

اختر Database session عندما تحتاج إبطالًا فوريًا، أو تسجيل خروج من كل الأجهزة، أو إدارة جلسات مركزية، أو Claims تتغير كثيرًا. استثمر في connection pooling وcaching واختبر سلوك ORM في runtime المستهدف.

### هل أستخدم Credentials؟

استخدمه عندما تملك سببًا تكامليًا واضحًا: نظام مستخدمين قديم أو API داخلي. لتطبيق جديد، OAuth/OIDC أو مزود هوية متخصص يقلل عبء password reset وMFA وcredential stuffing. هذا ليس تفضيلًا شكليًا؛ إنه تقليل لمساحة الهجوم والمسؤولية التشغيلية. [10]

### هل أضع التحقق في Proxy؟

ضع في Proxy قرارًا مبكرًا وخفيفًا، مثل منع زائر من الوصول إلى مجموعة صفحات عامة. ضع التحقق النهائي في كل صفحة وRoute Handler قبل البيانات أو التغييرات. Proxy ليس طبقة تفويض وحيدة.

---

## ✅ قائمة تسليم قبل الإنتاج

- [ ] `AUTH_SECRET` ثابت وعشوائي وموجود في Secret Manager.
- [ ] لا توجد أسرار في Git أو Client bundle.
- [ ] كل OAuth Callback URLs مضبوطة لكل بيئة.
- [ ] صفحات الدخول والخطأ موجودة ومترجمة وقابلة للوصول.
- [ ] كل Route Handler يفحص Authentication وAuthorization.
- [ ] لا توجد بيانات حساسة في `session` أو `jwt` callback.
- [ ] `redirect` يمنع Open Redirect.
- [ ] Credentials محمية بـrate limiting ورسائل عامة وHash مناسب.
- [ ] الجلسة منتهية وإبطالها وتسجيل الخروج مختبرة.
- [ ] Access Token rotation وrace conditions مفهومة ومختبرة.
- [ ] Adapter وORM متوافقان مع runtime والنشر.
- [ ] `npm run lint`, `npx tsc --noEmit`, و`npm run build` ناجحة.
- [ ] Logs لا تعرض Tokens أو كلمات مرور.
- [ ] تباين الألوان، لوحة المفاتيح، Labels، وFocus مجربة يدويًا.
- [ ] التبعيات مثبتة ومراقبة أمنيًا.
- [ ] خطة للتعامل مع تغير Auth.js أو الانتقال إلى Better Auth موجودة.

---

## 🚀 خطة انتقال من مبتدئ إلى محترف

### المستوى الأول: الأساسيات

نفذ تطبيقًا بـGitHub فقط، وافهم `auth.ts`, Route Handler, `auth()`, Redirect، وCookie session.

### المستوى الثاني: التطبيق العملي

أضف Google، صفحة دخول مخصصة، `useSession`, `SessionProvider`, وRoute Handler محميًا.

### المستوى الثالث: البيانات والصلاحيات

أضف Prisma Adapter، Database session، `role`، Module Augmentation، ودوال permissions قابلة للاختبار.

### المستوى الرابع: الإنتاج

أضف Events وAudit Logs، Refresh Token Rotation، Rate Limiting، Error Handling، Edge compatibility، CI، E2E، وObservability.

### المستوى الخامس: المعمارية

قارن Auth.js وBetter Auth وفق حالة مشروعك، صمم حدود الثقة، افصل AuthN عن AuthZ، واكتب ADR يوضح استراتيجية الجلسة والمزودين وتكلفة الهجرة.

---

## 🔗 المراجع

[1]: https://authjs.dev/getting-started/installation "Auth.js installation and environment setup"
[2]: https://github.com/nextauthjs/next-auth "Auth.js official GitHub repository and project status"
[3]: https://better-auth.com/docs/guides/next-auth-migration-guide "Better Auth migration guide for Auth.js"
[4]: https://nextjs.org/docs/app/guides/authentication "Next.js authentication concepts and guidance"
[5]: https://authjs.dev/getting-started/providers/passkey "Auth.js Passkey provider"
[6]: https://authjs.dev/getting-started/migrating-to-v5 "Auth.js NextAuth.js v5 upgrade guide"
[7]: https://authjs.dev/reference/nextjs/react "Auth.js React client API reference"
[8]: https://authjs.dev/getting-started/providers "Auth.js providers overview"
[9]: https://authjs.dev/guides/configuring-oauth-providers "Auth.js OAuth provider configuration"
[10]: https://authjs.dev/getting-started/providers/credentials "Auth.js Credentials provider"
[11]: https://authjs.dev/getting-started/providers/passkey "Auth.js WebAuthn and Passkeys setup"
[12]: https://authjs.dev/concepts/session-strategies "Auth.js session strategies"
[13]: https://authjs.dev/guides/extending-the-session "Auth.js extending the session"
[14]: https://authjs.dev/getting-started/adapters/prisma "Auth.js Prisma Adapter"
[15]: https://authjs.dev/guides/creating-a-database-adapter "Auth.js creating a database adapter"
[16]: https://authjs.dev/getting-started/migrating-to-v5 "Auth.js migration from NextAuth.js v4 to v5"
[17]: https://authjs.dev/guides/role-based-access-control "Auth.js role-based access control"
[18]: https://authjs.dev/guides/restricting-user-access "Auth.js restricting user access"
[19]: https://authjs.dev/guides/refresh-token-rotation "Auth.js refresh token rotation"
[20]: https://authjs.dev/guides/edge-compatibility "Auth.js edge compatibility"
[21]: https://www.w3.org/WAI/standards-guidelines/wcag/ "W3C Web Content Accessibility Guidelines"
[22]: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html "OWASP Authentication Cheat Sheet"

_تم إعداد هذا الدليل بالعربية مع الإبقاء على أسماء APIs والمصطلحات البرمجية بالإنجليزية لتسهيل نسخ الأمثلة ومقارنتها بالوثائق الرسمية._
