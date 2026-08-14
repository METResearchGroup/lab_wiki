# What data is available in Bluesky?

...

## Question 1: How do I get data about an account on Bluesky?

### Q1 Method 1: API

The AppView (the app layer that indexes repos and adds counts, search, and relationship metadata) is the easiest place to get a full picture of an account. Bluesky's own docs for this are in [Viewing profiles](https://docs.bsky.app/docs/tutorials/viewing-profiles). You pass a handle (e.g. `alice.bsky.social`) or a DID (e.g. `did:plc:...`). Auth is optional. Unauthenticated calls still return the public profile. Authenticated calls also fill in `viewer` fields that describe the logged-in account's relationship to that profile (following, blocking, muted, and so on).

Via the API, the appropriate endpoints that you'll need are:

- [`app.bsky.actor.getProfile`](https://docs.bsky.app/docs/api/app-bsky-actor-get-profile): one account. Query param is `actor` (handle or DID). Example: `GET https://public.api.bsky.app/xrpc/app.bsky.actor.getProfile?actor=bsky.app`
- [`app.bsky.actor.getProfiles`](https://docs.bsky.app/docs/api/app-bsky-actor-get-profiles): several accounts in one call. Query param is `actors` (repeat the param, one handle or DID per value). Same profile object as `getProfile`, just wrapped in a `profiles` array.

If you only have a handle and you need the DID first, use [`com.atproto.identity.resolveHandle`](https://docs.bsky.app/docs/api/com-atproto-identity-resolve-handle). If you also want who they follow or who follows them, that is a separate graph call (`app.bsky.graph.getFollows` / `app.bsky.graph.getFollowers`), not part of the profile object.

The schemas for these are in the AT Protocol lexicons:

- Endpoint: [`app.bsky.actor.getProfile`](https://github.com/bluesky-social/atproto/blob/main/lexicons/app/bsky/actor/getProfile.json)
- Response type: [`app.bsky.actor.defs#profileViewDetailed`](https://github.com/bluesky-social/atproto/blob/main/lexicons/app/bsky/actor/defs.json)

The fields you can expect, at time of writing, are the fields on `profileViewDetailed`. Required fields are only `did` and `handle`. Everything else can be missing.

Public identity and bio:

- `did`: stable account id
- `handle`: current username, e.g. `alice.bsky.social` (this can change)
- `displayName`: name shown on the profile
- `description`: bio text
- `pronouns`
- `website`
- `avatar`, `banner`: CDN URLs for the profile picture and banner
- `createdAt`: when the profile record was created
- `indexedAt`: when the AppView last indexed this profile

Counts and extras that the AppView computes (these are not stored as simple fields on the profile record in the PDS):

- `followersCount`, `followsCount`, `postsCount`
- `associated`: counts or flags for lists, feed generators, starter packs, whether the account is a labeler, and chat / activity-subscription settings
- `joinedViaStarterPack`: starter pack they joined through, if any
- `pinnedPost`: AT-URI + CID of the pinned post
- `labels`: moderation labels on the account
- `verification`: whether the account is verified, and by whom
- `status`: live / status overlay, if the account has one set

Only present (or only meaningful) when you are authenticated:

- `viewer`: muted / blocked / following / followed-by, plus related list and activity-subscription info for the logged-in account vs this profile

What this API does *not* give you, even though people often assume it does: the full list of posts, likes, or follows. Those live in other collections. `getProfile` is the indexed profile card, not a dump of the whole repo. For the raw profile record as stored on the PDS (`app.bsky.actor.profile`, rkey `self`), see Method 2 below. That record has display name, bio, pronouns, website, avatar/banner blobs, self-labels, pinned post, and `createdAt`. It does not have follower/follow/post counts.

### Q1 Method 2: PDS backfill

If you get the data directly from the PDS via `getRepo`, the following is available to you:

- ...

## Question 2: How do I get data about a post on Bluesky?

### Q2 Method 1: API

...

### Q2 Method 2: PDS backfill

- ...
