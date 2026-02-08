# CampusMeet Plan - Everything Explained

This document explains every technical choice in the plan in plain English. If you're a new grad and don't know what half this stuff means, this is for you.

---

## Table of Contents

1. [The Big Picture - How Web Apps Work](#1-the-big-picture)
2. [Why TanStack Start?](#2-why-tanstack-start)
3. [Why Cloudflare? (And What Even Is It?)](#3-why-cloudflare)
4. [The Database - D1 and Drizzle Explained](#4-the-database)
5. [Authentication - Google OAuth Explained](#5-authentication)
6. [KV Storage - What Is It and Why?](#6-kv-storage)
7. [The Database Tables - Why These 8?](#7-the-database-tables)
8. [File-Based Routing - What Are All These Files?](#8-file-based-routing)
9. [Server Functions - Frontend Meets Backend](#9-server-functions)
10. [The Smart Scheduling Algorithm](#10-the-smart-scheduling-algorithm)
11. [The UI Architecture](#11-the-ui-architecture)
12. [Tailwind CSS](#12-tailwind-css)
13. [The Config Files - What Do They Do?](#13-the-config-files)
14. [Free Tier Budget - Will I Get Charged?](#14-free-tier-budget)
15. [Glossary](#15-glossary)

---

## 1. The Big Picture

Every web app has three layers:

```
[Frontend]  <-->  [Backend]  <-->  [Database]
 (what users see)   (the logic)    (where data lives)
```

- **Frontend:** The HTML, CSS, and JavaScript that runs in the user's browser. It's the buttons, the feed, the forms.
- **Backend:** Code that runs on a server. It handles things like "is this user logged in?", "save this new event to the database", "fetch this user's Google Calendar".
- **Database:** Where all the data is permanently stored. Users, groups, events, comments - all saved here.

Traditionally, you'd need to set up and pay for a separate server (like a $5/month DigitalOcean droplet or an AWS instance). **We're avoiding that entirely** by using Cloudflare's free tier, which gives us all three layers for $0.

---

## 2. Why TanStack Start?

### What is it?

TanStack Start is a **fullstack TypeScript framework**. Let's break that down:

- **Fullstack** = it handles both frontend AND backend in one project. You don't need to build a separate API server.
- **TypeScript** = JavaScript but with types. Instead of `let name = "Fazil"` you write `let name: string = "Fazil"`. This catches bugs before your code even runs. If you try to do `name + 5`, TypeScript will yell at you.
- **Framework** = a pre-built structure with opinions about how to organize your code. Instead of wiring together 15 different libraries yourself, the framework does it for you.

### Why not Next.js or Remix?

You specifically asked for TanStack Start, but here's why it's a good choice anyway:

- **Next.js** is owned by Vercel and is slowly becoming more Vercel-specific. It works on Cloudflare but not as cleanly.
- **Remix** recently merged with React Router and is in a transitional state.
- **TanStack Start** has first-class Cloudflare Workers support. There's literally a `npm create cloudflare -- --framework=tanstack-start` command. It was designed to work on the edge (Cloudflare's servers around the world).

### What are "server functions"?

This is the magic of TanStack Start. Normally, to talk to a database, you'd need to:
1. Build a REST API with endpoints like `GET /api/groups`
2. Call that API from your frontend with `fetch('/api/groups')`
3. Parse the response

With server functions, you just write a function:

```ts
// This function ALWAYS runs on the server, never in the browser
export const getMyGroups = createServerFn({ method: 'GET' })
  .handler(async () => {
    return db.select().from(groups).all();
  });
```

Then in your React component, you call it like a normal function. TanStack Start handles all the network stuff automatically. The function runs on the server, the result gets sent to the browser. You never build REST endpoints manually.

### What is "file-based routing"?

Instead of writing code like "when the URL is `/groups`, show the GroupsPage component", you just create a file at `src/routes/groups/index.tsx` and it automatically becomes the `/groups` page.

The file structure IS the URL structure:
- `src/routes/index.tsx` -> `yourapp.com/`
- `src/routes/_authed/feed.tsx` -> `yourapp.com/feed`
- `src/routes/_authed/groups/new.tsx` -> `yourapp.com/groups/new`

---

## 3. Why Cloudflare?

### What is Cloudflare?

Cloudflare is a company that runs servers in 300+ cities worldwide. They're primarily known for protecting websites from attacks, but they also let you run code on their servers.

### What are "Workers"?

Cloudflare Workers are tiny programs that run on Cloudflare's servers. When a user visits your app, the request goes to the nearest Cloudflare server (maybe Dallas if you're at OU, or wherever is closest) and your code runs there.

Think of it like this: instead of having ONE server in Virginia that everyone connects to, you have copies of your app running in 300 cities. The user connects to whichever is closest. This makes your app fast.

### Why not just use a regular server?

- **Cost:** A regular server (AWS, DigitalOcean, etc.) costs $5-20/month minimum. Cloudflare Workers are free for 100,000 requests/day.
- **Scaling:** If your app goes viral on campus, a $5 server crashes. Workers auto-scale.
- **No maintenance:** You don't need to update the server's operating system, install security patches, etc.

### What is "the edge"?

"Edge" means "close to the user." When people say "edge computing," they mean running code on servers that are geographically close to whoever is using the app, instead of in one central data center. Cloudflare Workers run "on the edge."

### What is Wrangler?

Wrangler is Cloudflare's command-line tool. It's how you:
- Create databases (`wrangler d1 create my-database`)
- Deploy your app (`wrangler deploy`)
- Test locally (`wrangler dev`)

Think of it like `git` but for Cloudflare. You install it with `npm install wrangler`.

---

## 4. The Database

### What is D1?

D1 is Cloudflare's database. Under the hood, it's **SQLite** - the same database engine that runs on your phone (every iPhone and Android uses SQLite for app data).

Why SQLite and not PostgreSQL or MySQL?
- It's simpler. No separate database server to manage.
- Cloudflare manages it for you. You never SSH into a database server.
- Free tier gives you 5GB storage and 5 million reads/day. That's more than enough.

### What is Drizzle ORM?

An **ORM** (Object-Relational Mapper) lets you talk to the database using TypeScript instead of raw SQL.

Without Drizzle (raw SQL):
```sql
SELECT * FROM users WHERE email = 'fazil@example.com';
```

With Drizzle:
```ts
const user = await db.select().from(users).where(eq(users.email, 'fazil@example.com'));
```

Why is this better?
- **Type safety.** If you misspell `email` as `emal`, TypeScript catches it immediately.
- **Auto-complete.** Your editor knows all the column names and suggests them.
- **Migrations.** When you change your schema (add a column, rename a table), Drizzle generates the SQL migration files for you.

### What is a "migration"?

A migration is a set of SQL commands that change your database structure. When you add a new table or column, you need to tell the database about it.

Workflow:
1. You edit `schema.ts` (add a new table)
2. Run `npx drizzle-kit generate` (Drizzle compares your schema to the database and generates a SQL file like `0001_add_groups_table.sql`)
3. Run `wrangler d1 migrations apply campusmeet-db --local` (applies the SQL to your local dev database)
4. When you deploy, run the same command with `--remote` to update the production database

### What is nanoid?

Every row in a database needs a unique ID. There are two approaches:
- **Auto-increment:** 1, 2, 3, 4... Simple but reveals info (user #5 knows there are only 4 users before them, and can guess that `/groups/2` exists)
- **nanoid:** Random strings like `V1StGXR8_Z5jdHi6B-myT`. Impossible to guess. URL-safe.

We use nanoid for all IDs.

---

## 5. Authentication

### What is OAuth 2.0?

OAuth is the system behind "Sign in with Google" buttons. Here's how it works in plain English:

1. User clicks "Sign in with Google" on your app
2. Your app redirects them to Google's login page
3. User enters their Google credentials (on Google's site, never on yours)
4. Google says "hey, this user is fazil@gmail.com, here's a temporary code"
5. Your server takes that code and exchanges it for **tokens** (secret keys)
6. You use those tokens to access Google APIs (like Calendar) on behalf of the user

**Why Google OAuth instead of username/password?**
- You never store passwords. Passwords are a security nightmare.
- Every college student has a Google account.
- You get their name and profile photo for free.
- You get permission to access their Google Calendar (which is the whole point of the app).

### What are tokens?

When Google gives you tokens after sign-in, you get:
- **Access token:** A temporary key (expires in 1 hour) that lets you call Google APIs like Calendar
- **Refresh token:** A long-lived key that lets you get a new access token when the old one expires
- **ID token:** Contains the user's basic info (email, name, photo) encoded as a JWT

### What is a session?

After the user logs in, you need to remember that they're logged in. A **session** is just a cookie (a small piece of data stored in the browser) that says "this browser belongs to user X."

Our flow:
1. User logs in via Google
2. We generate a random session token (like `abc123xyz`)
3. We store `session:abc123xyz -> { userId: "fazil" }` in KV (Cloudflare's key-value store)
4. We set a cookie in the user's browser: `session=abc123xyz`
5. On every request, the browser sends this cookie. We look up `session:abc123xyz` in KV, find `{ userId: "fazil" }`, and know who they are.

### What is middleware?

Middleware is code that runs BEFORE your page/function logic. Think of it as a bouncer at a club:

```
User request -> [Auth Middleware: "Are you logged in?"] -> [Your page code]
                      |
                      v (if not logged in)
                  Redirect to login page
```

Every protected page goes through the auth middleware first. If the user isn't logged in, they get redirected. If they are, the middleware passes the `userId` to the page so it knows who's asking.

---

## 6. KV Storage

### What is KV?

KV stands for **Key-Value store**. It's the simplest possible database. You store data like:

```
Key: "session:abc123"  ->  Value: '{"userId": "fazil"}'
Key: "oauth:access:fazil"  ->  Value: "ya29.a0AfH6SMB..."
```

That's it. No tables, no columns, no SQL. Just keys and values. Like a giant dictionary.

### Why use KV instead of D1 for sessions and tokens?

- **TTL (Time To Live):** KV lets you set an expiration. "Store this access token, but delete it automatically after 3600 seconds." D1 can't do this - you'd need to write a cleanup job.
- **Speed:** KV is cached at the edge (nearest Cloudflare server). D1 queries are slightly slower.
- **Simplicity:** Session lookup is always "give me the value for this key." You don't need SQL for that.

### Why not use KV for everything?

KV has no relationships. You can't say "give me all events in group X, sorted by date, with the creator's name." That requires SQL joins, which only D1 can do.

**Rule of thumb:** KV for simple lookups (sessions, tokens). D1 for everything with structure and relationships.

---

## 7. The Database Tables

### Why 8 tables? Why these specific ones?

Let me walk through each:

**`users`** - You need to know who your users are. Stores their Google profile info.

**`groups`** - The core social unit. A group is a friend circle. Has an invite code so people can join (like a Discord invite link).

**`group_members`** - This is called a **junction table** (or join table). It connects users to groups. One user can be in many groups. One group can have many users. This "many-to-many" relationship needs its own table. It also stores the role (admin vs member).

**`events`** - The main feature. An event belongs to a group and has a status lifecycle:
- `polling` -> people are still voting on times
- `finalized` -> a time has been picked
- `cancelled` -> event was called off

**`event_time_proposals`** - When you create an event, the smart scheduler suggests time slots. Each suggestion is a row in this table. The `source` column tells you if it was auto-generated by the algorithm (`smart`) or manually proposed by the creator (`manual`).

**`event_votes`** - Users vote on proposals. "I can do Tuesday 3pm" (yes), "I could make Wednesday work if needed" (if_needed), "Thursday is a no-go" (no).

**`event_rsvps`** - After the time is finalized, users RSVP: going, maybe, not going. This is separate from voting because voting is about WHEN, RSVPs are about WHETHER.

**`feed_posts`** - Comments and updates on events. "Who's bringing snacks?" or "Location changed to the library." This makes it social.

**`reactions`** - Emoji reactions on feed posts. Like Instagram or Slack reactions. The uniqueness constraint (`postId + userId + emoji`) means you can't spam the same reaction.

### What is a "junction table"?

When two things have a many-to-many relationship, you need a table in between:

```
User A is in Group 1, Group 2
User B is in Group 1, Group 3
Group 1 has User A, User B

You can't store this in either the users or groups table.
So you make group_members:

| userId | groupId |
|--------|---------|
| A      | 1       |
| A      | 2       |
| B      | 1       |
| B      | 3       |
```

---

## 8. File-Based Routing

### The underscore prefix: `_authed`

Files/folders starting with `_` are **layout routes**. They don't create a URL themselves. `_authed.tsx` is a wrapper that checks "is the user logged in?" before showing any child routes.

So `_authed/feed.tsx` creates the URL `/feed` (not `/_authed/feed`). The `_authed` part is invisible in the URL - it just means "this page requires login."

### The dollar sign: `$groupId`

`$groupId` is a **dynamic segment**. It matches any value:
- `/groups/abc123` -> `$groupId = "abc123"`
- `/groups/xyz789` -> `$groupId = "xyz789"`

Inside the component, you read `$groupId` with a hook and use it to fetch the right group from the database.

### The `__root.tsx` file

This is the root layout. It wraps EVERY page. It contains:
- The `<html>` and `<body>` tags
- The `<head>` with meta tags (title, viewport for mobile, theme color)
- Links to fonts and the PWA manifest

### The `api/auth/` folder

These routes don't render pages. They're server-only endpoints that handle the OAuth flow. When Google redirects back to your app after login, it goes to `/api/auth/callback`, which is a server function that exchanges the code for tokens and creates the session.

---

## 9. Server Functions

### How they're organized

Each file in `app/src/server/` handles one domain:

- **`auth.ts`** - Login, logout, session management. Talks to Google OAuth.
- **`middleware.ts`** - The bouncer. Checks every authenticated request.
- **`groups.ts`** - Create group, join group, list groups, group details.
- **`events.ts`** - Create event, finalize time, cancel, list events.
- **`scheduling.ts`** - The smart scheduling algorithm. Talks to Google Calendar API.
- **`feed.ts`** - Feed aggregation, comments, reactions.
- **`rsvp.ts`** - Set/change RSVPs.
- **`voting.ts`** - Vote on time proposals.

### What does "validator" do?

In the plan, you'll see `.validator(...)` on server functions. This checks that the data sent from the frontend is correct BEFORE the function runs.

```ts
.validator((data: { name: string; description?: string }) => data)
```

This says "I expect an object with a required `name` string and an optional `description` string." If the frontend sends something else (like `{ name: 123 }`), the function rejects it.

---

## 10. The Smart Scheduling Algorithm

This is the most interesting part of the app. Here's how it works step by step:

### Step 1: User creates an event
They fill out:
- Title: "Study session"
- Duration: 90 minutes
- Date range: "Sometime between Mon Feb 10 - Fri Feb 14"
- Preferred hours: "Between 10am and 8pm"

### Step 2: Fetch everyone's busy times
The server calls Google Calendar's **FreeBusy API** for each group member who connected their calendar. This returns their busy blocks:

```
Fazil: Busy Mon 9am-11am, Mon 2pm-3pm, Tue 1pm-4pm, ...
Sarah: Busy Mon 10am-12pm, Wed all day, ...
Jake:  Busy Mon 9am-10am, Tue 3pm-5pm, ...
```

Note: We only see WHEN they're busy, not WHAT they're doing. Privacy is preserved.

### Step 3: Invert busy to free
For each person, we flip the busy times into free times within the preferred window:

```
Fazil's free times (10am-8pm window):
  Mon: 11am-2pm, 3pm-8pm
  Tue: 10am-1pm, 4pm-8pm
  ...
```

### Step 4: Find overlapping free times
We intersect everyone's free times to find when EVERYONE (or most people) are free:

```
All 3 free: Mon 11am-12pm, Tue 10am-1pm
2 of 3 free: Mon 3pm-8pm, ...
```

### Step 5: Extract time slots
From those free windows, we extract slots matching the event duration (90 min):

```
Mon 11am-12:30pm (all 3 free)
Tue 10am-11:30am (all 3 free)
Tue 10:30am-12pm (all 3 free)
Tue 11am-12:30pm (all 3 free)
Mon 3pm-4:30pm (2 of 3 free)
...
```

### Step 6: Score and rank
Each slot gets a score based on:
- How many people can make it (most important)
- Time of day (evening/afternoon preferred for social events)
- Weekend bonus (people are more free on weekends)

### Step 7: Present top 8
The best 8 options are saved and shown to the group for voting.

### Step 8: Vote and finalize
Group members vote (yes / if needed / no) on each proposal. The event creator picks the winning time. Once finalized, the event is automatically added to everyone's Google Calendar.

### Why FreeBusy API instead of reading full calendar events?

**Privacy.** The FreeBusy API only tells you "busy from 2pm-3pm." It doesn't tell you "Doctor's appointment" or "Therapy session." Users don't want their friends seeing their calendar event titles.

---

## 11. The UI Architecture

### What is "mobile-first"?

It means we design for phones first, then adapt for bigger screens. Why?
- College students use their phones for everything
- It's easier to scale up (phone -> tablet -> desktop) than scale down
- If it works on a small screen, it definitely works on a big one

### What is an "AppShell"?

The AppShell is the outer frame of the app that stays the same on every page:
- On **mobile**: A bottom navigation bar (like Instagram/TikTok) with 5 tabs
- On **desktop**: A left sidebar with navigation links

The content area in the middle changes as you navigate between pages, but the shell stays.

### What is a "bottom sheet"?

Instead of a popup/modal that floats in the center of the screen (awkward on mobile), a bottom sheet slides up from the bottom. It's the pattern you see in Google Maps, Apple Maps, Uber, etc. Much more thumb-friendly.

### What is a "component"?

A component is a reusable piece of UI. Think of it like LEGO blocks:

- `Button` - a styled button you use everywhere
- `Card` - a rounded white box with a shadow
- `Avatar` - a circular profile picture
- `GroupCard` - a card showing a group's emoji, name, and member count

You build small components and combine them into pages. This means if you want to change how all buttons look, you change ONE file, not 50.

### What is "optimistic update"?

When you tap "Going" on an RSVP, the button should change immediately. You don't want to wait 500ms for the server to respond. An optimistic update means:

1. User taps "Going"
2. UI instantly shows "Going" (before the server responds)
3. Server request happens in the background
4. If it succeeds: do nothing (UI is already correct)
5. If it fails: revert the UI and show an error toast

This makes the app feel instant.

---

## 12. Tailwind CSS

### What is it?

Tailwind is a CSS framework where you style elements using class names directly in your HTML:

```html
<!-- Without Tailwind -->
<div style="padding: 16px; background: white; border-radius: 12px; box-shadow: 0 1px 2px rgba(0,0,0,0.05);">

<!-- With Tailwind -->
<div class="p-4 bg-white rounded-xl shadow-sm">
```

Each class does one thing:
- `p-4` = padding: 16px
- `bg-white` = background: white
- `rounded-xl` = border-radius: 12px
- `shadow-sm` = subtle box shadow
- `text-indigo-600` = text color: indigo

### Why Tailwind instead of regular CSS?

- No naming things. You never think "what should I call this class?"
- No switching between files. Styles are right there in the component.
- Consistent spacing/colors. Tailwind's defaults look good.
- Tree-shaking. Only the classes you actually use end up in the final CSS (tiny file size).

### What are breakpoints?

Tailwind uses responsive prefixes:
- No prefix = mobile (default)
- `sm:` = 640px+ (large phones)
- `md:` = 768px+ (tablets)
- `lg:` = 1024px+ (desktops)

Example:
```html
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3">
```
This means: 1 column on mobile, 2 columns on tablet, 3 columns on desktop.

---

## 13. The Config Files

### `vite.config.ts` - The build tool config

**Vite** is the build tool that turns your TypeScript + React code into optimized JavaScript that browsers can run. This file tells Vite what plugins to use:

```ts
plugins: [
  cloudflare(),     // Makes the build work on Cloudflare Workers
  tanstackStart(),  // Adds TanStack Start's server function magic
  viteReact(),      // Adds React JSX support
]
```

**Order matters.** Cloudflare must be first because it needs to wrap everything else.

### `wrangler.jsonc` - Cloudflare config

This tells Cloudflare about your project:
- What your app is called
- What databases (D1) and storage (KV) it uses
- What compatibility flags it needs (`nodejs_compat` = "let me use Node.js APIs")
- Environment variables (like your Google OAuth credentials)

### `drizzle.config.ts` - ORM config

Tells Drizzle:
- Where your schema file is
- What type of database you're using (SQLite / D1)
- Where to put migration files
- How to connect to the database for generating migrations

### `tsconfig.json` - TypeScript config

Tells TypeScript:
- What JavaScript version to target
- How strict to be with type checking
- Where to find your source files

### `package.json` - Project manifest

Lists:
- All your dependencies (libraries your code needs)
- Scripts (shortcuts like `npm run dev` to start the dev server)
- Project metadata (name, version)

---

## 14. Free Tier Budget

### Will I get charged?

**No**, as long as you stay within the free tier limits. Cloudflare's free tier is genuinely free - no credit card required to start. Here's what you get:

| What | Free Limit | What it means |
|------|-----------|---------------|
| Workers requests | 100,000/day | Each time someone loads a page or makes an action = 1 request. 100K/day means ~500 users can each load 200 pages/day. |
| D1 reads | 5,000,000/day | Each database query = 1+ reads. Loading a page might do 3-4 reads. 5M is very generous. |
| D1 writes | 100,000/day | Each time data is saved (new comment, RSVP, etc.) = 1 write. 100K is plenty. |
| D1 storage | 5 GB | Text data is tiny. 500 users with all their events/comments would use maybe 50MB. |
| KV reads | 100,000/day | Each page load checks the session = 1 KV read. This is the tightest limit. |
| KV writes | 1,000/day | New sessions and token refreshes. ~200/day expected. |

### What if it gets popular?

If you exceed free limits, Cloudflare doesn't charge you by surprise - your app just stops working until the next day (limits reset at midnight UTC). If you need more, Cloudflare's paid plan ($5/month) gives you 10x the limits.

### The KV optimization

The tightest limit is KV reads (100K/day) because every single page load needs one to check the session cookie. If this becomes a problem, we can switch to **signed JWT cookies**:

Instead of storing sessions in KV and looking them up, we encode the session data directly into the cookie and sign it with a secret key. The server can verify the signature without any database lookup. This eliminates KV reads for session checks entirely.

---

## 15. Glossary

| Term | Meaning |
|------|---------|
| **API** | Application Programming Interface. A way for programs to talk to each other. Google Calendar API = a set of URLs your server can call to read/write calendar data. |
| **Bearer token** | An access token sent in HTTP headers like `Authorization: Bearer ya29.abc...` to prove you have permission. |
| **Cookie** | A small piece of data stored in the browser. Sent automatically with every request to the same domain. Used for sessions. |
| **CRUD** | Create, Read, Update, Delete. The four basic database operations. |
| **Deploy** | Push your code to production (make it live on the internet). |
| **Edge** | Servers close to the user geographically. |
| **Endpoint** | A specific URL that does something on the server (e.g., `/api/auth/callback`). |
| **Environment variable** | A secret value stored outside your code (like `GOOGLE_CLIENT_SECRET`). Never committed to git. |
| **Fetch** | The browser/server function for making HTTP requests. `fetch('https://api.google.com/...')` |
| **FreeBusy API** | A Google Calendar endpoint that returns when someone is busy/free without revealing event details. |
| **HMAC-SHA256** | A cryptographic algorithm for signing data. Used to verify that a cookie hasn't been tampered with. |
| **HTTP** | The protocol web browsers use to communicate with servers. GET = read, POST = create/submit. |
| **IDE** | Integrated Development Environment. VS Code, Cursor, etc. |
| **JSON** | JavaScript Object Notation. The standard format for data exchange: `{"name": "Fazil", "age": 22}` |
| **JSX/TSX** | A syntax that lets you write HTML-like code inside JavaScript/TypeScript. React uses this. |
| **JWT** | JSON Web Token. A signed, encoded string containing data (like user ID). Can be verified without a database lookup. |
| **Layout route** | A wrapper component that surrounds child routes. `_authed.tsx` wraps all authenticated pages. |
| **Migration** | A SQL script that changes database structure (add table, add column, etc.). |
| **Mutation** | A request that changes data (create, update, delete). Opposite of a query (read). |
| **ORM** | Object-Relational Mapper. Lets you interact with a database using programming language objects instead of raw SQL. |
| **PWA** | Progressive Web App. A website that can be "installed" on your phone's home screen and feels like a native app. |
| **REST** | A pattern for APIs where URLs represent resources (GET `/groups` = list groups, POST `/groups` = create group). |
| **Scaffold** | Generate the initial project structure with boilerplate code. |
| **Schema** | The structure of your database - what tables exist, what columns each has, what types they are. |
| **Scope** | In OAuth, the permissions you're requesting. `calendar.readonly` = "let me see your calendar." |
| **SSR** | Server-Side Rendering. The server generates the HTML instead of the browser. Makes the initial page load faster. |
| **TTL** | Time To Live. How long before data expires and is auto-deleted. |
| **Type-safe** | The compiler catches mistakes like passing a number where a string is expected, before your code runs. |
