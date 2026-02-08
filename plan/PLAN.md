# CampusMeet - Social Meetup App Plan

## Context

A campus social app for college students to plan meetups with friends. The core differentiator is **smart scheduling** - the app pulls everyone's Google Calendar availability and finds the best overlapping times. Built on a $0 budget using Cloudflare's free tier.

**Target:** 100-500 users on a single campus. Built inside the current repo at `app/`.

## Tech Stack

- **Fullstack framework:** TanStack Start (TypeScript, file-based routing, server functions)
- **Runtime:** Cloudflare Workers (100K requests/day free)
- **Database:** Cloudflare D1 (SQLite, 5M reads/day, 100K writes/day free)
- **ORM:** Drizzle ORM (sqlite-core dialect, D1 adapter)
- **Sessions/Tokens:** Cloudflare KV (100K reads/day free)
- **Auth:** Google OAuth 2.0 (manual flow via `fetch`, no SDK)
- **Calendar:** Google Calendar REST API (FreeBusy + Events endpoints)
- **Styling:** Tailwind CSS (mobile-first)
- **Deployment:** Wrangler CLI

## Database Schema (D1 + Drizzle)

**File: `app/src/db/schema.ts`** - 8 tables:

| Table | Purpose | Key columns |
|-------|---------|-------------|
| `users` | Google profile data | `id (nanoid)`, `googleId`, `email`, `displayName`, `avatarUrl`, `calendarConnected` |
| `groups` | Invite-only friend groups | `id`, `name`, `emoji`, `inviteCode (8-char)`, `creatorId` |
| `group_members` | Users <-> Groups junction | `groupId`, `userId`, `role (admin/member)` |
| `events` | Meetup events | `id`, `title`, `location`, `groupId`, `status (polling/finalized/cancelled)`, `finalizedTime`, `durationMinutes`, `pollStartDate`, `pollEndDate` |
| `event_time_proposals` | Suggested time slots | `eventId`, `startTime`, `endTime`, `source (smart/manual)`, `score` |
| `event_votes` | Votes on time proposals | `proposalId`, `userId`, `vote (yes/if_needed/no)` |
| `event_rsvps` | Final attendance | `eventId`, `userId`, `status (going/maybe/not_going)` |
| `feed_posts` | Comments/updates on events | `eventId`, `authorId`, `content`, `type (comment/update)` |
| `reactions` | Emoji reactions on posts | `postId`, `userId`, `emoji` |

**KV storage** (not D1): OAuth access tokens (with TTL), refresh tokens, session data. Keys: `oauth:access:{userId}`, `oauth:refresh:{userId}`, `session:{token}`.

## Route Structure

```
app/src/routes/
  __root.tsx                          # HTML shell, head, scripts
  index.tsx                           # Landing page / sign-in
  _authed.tsx                         # Auth guard layout (checks session, redirects)
  _authed/
    feed.tsx                          # Dashboard: aggregated activity feed
    profile.tsx                       # Edit profile, calendar settings
    calendar.tsx                      # Google Calendar connection management
    groups/
      index.tsx                       # List user's groups
      new.tsx                         # Create group form
      join.tsx                        # Join via invite code
      $groupId/
        index.tsx                     # Group detail: members + events
        settings.tsx                  # Edit group, manage members
        events/
          new.tsx                     # Create event + smart scheduling
    events/
      $eventId/
        index.tsx                     # Event detail: feed, RSVPs, comments
        vote.tsx                      # Vote on time proposals
        schedule.tsx                  # Smart scheduling results
  api/auth/
    google.tsx                        # Initiate Google OAuth redirect
    callback.tsx                      # Handle OAuth callback
    logout.tsx                        # Clear session
```

## Server Functions

**File: `app/src/server/`** - organized by domain:

| File | Key functions |
|------|--------------|
| `auth.ts` | `handleOAuthCallback`, `getSession`, `getCurrentUser`, `logout` |
| `middleware.ts` | `authMiddleware` - validates session cookie via KV, injects `userId` |
| `groups.ts` | `createGroup`, `joinGroup`, `getMyGroups`, `getGroupDetail`, `leaveGroup` |
| `events.ts` | `createEvent`, `getEventDetail`, `getGroupEvents`, `finalizeEventTime` |
| `scheduling.ts` | `runSmartScheduling`, `syncEventToCalendar` |
| `feed.ts` | `getFeedData`, `createComment`, `toggleReaction` |
| `rsvp.ts` | `setRsvp` |
| `voting.ts` | `voteOnProposal` |

## Smart Scheduling Algorithm

**File: `app/src/server/scheduling.ts`**

1. Creator specifies: date range, preferred hours (e.g. 10am-8pm), event duration
2. For each group member with calendar connected: fetch Google Calendar FreeBusy data
3. Invert busy periods into free slots within preferred hours
4. Intersect all members' free slots to find common windows
5. Extract fixed-duration candidate slots (30-min step)
6. Score each slot: `(available_members / total) * 100` + time-of-day bonus (evening +15, afternoon +10) + weekend bonus (+5)
7. Store top 8 proposals in `event_time_proposals` table
8. Group votes on proposals, creator finalizes
9. On finalize: create Google Calendar event with attendees via Calendar API

## UI Architecture (Mobile-First)

**Layout:** `AppShell` with bottom nav (mobile) / sidebar (desktop)
- 5 tabs: Feed, Groups, +Create, Calendar, Profile
- Center "Create" button opens bottom sheet (New Group / New Event)

**Component structure:**
```
app/src/components/
  layout/    AppShell, BottomNav, SideNav, PageHeader
  ui/        Button, Card, Input, Badge, Avatar, AvatarStack, Modal, Skeleton, Toast
  features/
    groups/      GroupCard, InviteCodeDisplay, MemberList
    events/      EventCard, RsvpButtons, TimeProposalCard, VotingInterface
    scheduling/  SmartScheduleForm, AvailabilityGrid, ProposalRanking
    feed/        FeedItem, CommentInput, CommentThread, ReactionBar
```

**Color scheme:** Indigo-600 primary, Gray-50 background, white cards with rounded-xl + shadow-sm

## Implementation Phases

### Phase 1: Scaffolding + Auth (first)
- `npm create cloudflare@latest app -- --framework=tanstack-start`
- Configure `vite.config.ts` (cloudflare plugin FIRST, then tanstackStart, then react)
- Configure `wrangler.jsonc` with D1 binding, KV namespace, `nodejs_compat` flag
- Install: `drizzle-orm`, `drizzle-kit`, `nanoid`, `tailwindcss`, `@tailwindcss/vite`
- Create `users` table, run first migration
- Implement Google OAuth flow (redirect -> callback -> upsert user -> session cookie)
- Build landing page with Google Sign-In button
- Build `_authed.tsx` layout guard + basic feed page showing "Welcome, {name}"

### Phase 2: Groups + Invites
- Add `groups`, `group_members` tables + migration
- Server functions: createGroup, joinGroup, getMyGroups, getGroupDetail
- Routes: group list, create group, join via code, group detail page
- Components: GroupCard, InviteCodeDisplay, MemberList

### Phase 3: Events + Feed
- Add remaining 5 tables + migration
- Server functions: createEvent, RSVP, voting, comments, reactions, feed aggregation
- Routes: event creation, event detail, voting, feed dashboard
- Components: EventCard, RsvpButtons, TimeProposalCard, CommentThread, ReactionBar

### Phase 4: Smart Scheduling + Calendar
- Implement scheduling algorithm (FreeBusy API, slot intersection, scoring)
- Implement calendar sync (create Google Calendar events on finalize)
- Routes: scheduling results page, calendar settings
- Components: SmartScheduleForm, AvailabilityGrid, ProposalRanking

### Phase 5: Polish + PWA
- Optimistic updates on mutations
- Loading skeletons and error boundaries
- PWA manifest + service worker for installability
- Touch-friendly interactions, bottom sheets for modals

## Key Config Files

**`app/vite.config.ts`:**
```ts
plugins: [
  cloudflare({ viteEnvironment: { name: 'ssr' } }),  // MUST be first
  tanstackStart(),
  viteReact(),
]
```

**`app/wrangler.jsonc`:** needs `"compatibility_flags": ["nodejs_compat"]`, D1 binding as `DB`, KV binding as `KV`

**`app/drizzle.config.ts`:** dialect `sqlite`, driver `d1-http`

## Free Tier Budget (500 users)

| Resource | Limit | Est. usage | Status |
|----------|-------|-----------|--------|
| Workers requests | 100K/day | ~50K | OK |
| D1 reads | 5M/day | ~200K | OK |
| D1 writes | 100K/day | ~5K | OK |
| D1 storage | 5GB | ~50MB | OK |
| KV reads | 100K/day | ~50K | Tight* |

*If KV reads become tight: switch sessions to signed JWT cookies (HMAC-SHA256), eliminating 1 KV read per request. Keep KV only for OAuth tokens.

## Verification

1. **Auth:** Sign in with Google -> see your name on feed -> sign out -> redirected to landing
2. **Groups:** Create group -> copy invite code -> open incognito -> sign in as different account -> join with code -> see both members
3. **Events:** Create event in group -> other member sees it -> both RSVP -> add comments -> react
4. **Smart scheduling:** Create event with "smart schedule" -> algorithm returns ranked times -> vote -> finalize -> check Google Calendar for the event
5. **Deploy:** `npm run deploy` from `app/` -> verify on `*.workers.dev` domain
