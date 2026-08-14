# What data is available in Bluesky?

What data can we expect to get in Bluesky and what's generally not available or has to be computed on your end? Here's a brief primer.

**Note on AI usage**: AI was used to create these docs. However, it references specific documentation pages and was cross-checked by Mark, after manually writing all the other docs in this folder.

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

`getRepo` downloads the account's repository as a CAR file (a binary archive of every current public record in that repo). How to call it, and how the relay redirects to the right PDS, is in [How do we get Bluesky data?](HOW_DO_WE_GET_BLUESKY_DATA.md). You can also pull one collection at a time with `com.atproto.repo.listRecords`, or one record with `com.atproto.repo.getRecord`. The repo is the current public state. Deleted records are gone.

The important constraint is that a repo only stores what *this* account wrote. Alice's repo has Alice's profile, Alice's posts, the accounts Alice follows, and the likes Alice made. It does not have Alice's followers, likes on Alice's posts, or any of the AppView counts.

If you get the data directly from the PDS via `getRepo`, the following is available to you:

- **`app.bsky.actor.profile`** (rkey is always `self`): the raw profile record. Fields, at time of writing, are `displayName`, `description`, `pronouns`, `website`, `avatar` and `banner` (blob refs, not CDN URLs), `labels` (self-labels only), `joinedViaStarterPack`, `pinnedPost`, and `createdAt`. Schema: [`app.bsky.actor.profile`](https://github.com/bluesky-social/atproto/blob/main/lexicons/app/bsky/actor/profile.json).
- **`app.bsky.actor.status`** (rkey `self`): live / status overlay, if they set one.
- **`app.bsky.actor.contentVisibilityDeclaration`** (rkey `self`): whether they asked to be hidden from algorithmic recommendations.
- **`app.bsky.graph.follow`**: who *they* follow. Each record is `{ subject: <DID>, createdAt }`. This is how you get a follow list from a backfill. You do *not* get who follows them.
- **`app.bsky.graph.block`**: who they block. Blocks are public on Bluesky.
- **`app.bsky.graph.list`**, **`listitem`**, **`listblock`**: lists they created, members of those lists, and whole-list blocks.
- **`app.bsky.graph.starterpack`**: starter packs they created.
- **`app.bsky.graph.verification`**: verification records *they issued* (only meaningful if the issuer is a trusted verifier).
- **`app.bsky.feed.generator`**: custom feeds they publish.
- **`app.bsky.labeler.service`** (rkey `self`): present if the account is a labeler.
- Their own **`app.bsky.feed.post`**, **`like`**, and **`repost`** records (activity they authored). See Question 2.
- **Blobs** (avatars, banners, images): not inside the CAR. Use `com.atproto.sync.listBlobs` / `getBlob`. See [Download and parse repository exports](https://atproto.com/blog/repo-export).

What you will *not* find in this repo, even though `getProfile` returns them:

- `handle` (that lives in the DID document / PLC directory, and can change)
- `followersCount`, `followsCount`, `postsCount` (you can *count* follows and posts in this repo, but follower count requires other people's repos)
- `followers` as a list (those are `follow` records in *other* repos whose `subject` is this DID)
- AppView `labels` from third-party labelers
- `viewer` relationship fields
- mutes, bookmarks, and app preferences (those are private and are not in the public repo)

The official note from Bluesky is the same: [repo export](https://atproto.com/blog/repo-export) includes posts and likes, and does not include mutes or private list subscriptions.

## Question 2: How do I get data about a post on Bluesky?

### Q2 Method 1: API

A Bluesky post is identified by its AT-URI, which looks like `at://did:plc:alice123/app.bsky.feed.post/3kxyz`. That URI has the author's DID, the collection (`app.bsky.feed.post`), and the record key. Bluesky's own docs for threads are in [Viewing threads](https://docs.bsky.app/docs/tutorials/viewing-threads). Auth is optional. Unauthenticated calls still return the public post. Authenticated calls also fill in `viewer` fields (whether you liked, reposted, or bookmarked it, and whether replies or embeds are disabled for you).

Via the API, the appropriate endpoints that you'll need are:

- [`app.bsky.feed.getPosts`](https://docs.bsky.app/docs/api/app-bsky-feed-get-posts): one or more posts, by AT-URI. Query param is `uris` (repeat the param, max 25). Example: `GET https://public.api.bsky.app/xrpc/app.bsky.feed.getPosts?uris=at://did:plc:z72i7hdynmk6r22z27h6tvur/app.bsky.feed.post/3l6oveex3ii2l`
- [`app.bsky.feed.getPostThread`](https://docs.bsky.app/docs/api/app-bsky-feed-get-post-thread): one post plus its parents and replies. Query param is `uri`. Optional `depth` (reply levels, default 6) and `parentHeight` (parent levels, default 80). A node in the tree can be a real post, a `notFoundPost` (deleted or taken down), or a `blockedPost`.

If you want every post by an account rather than one known URI, use [`app.bsky.feed.getAuthorFeed`](https://docs.bsky.app/docs/api/app-bsky-feed-get-author-feed). If you want who liked or reposted a given post, that is a separate call (`app.bsky.feed.getLikes` / `app.bsky.feed.getRepostedBy`), not part of the post object itself.

The schemas for these are in the AT Protocol lexicons:

- Endpoints: [`app.bsky.feed.getPosts`](https://github.com/bluesky-social/atproto/blob/main/lexicons/app/bsky/feed/getPosts.json), [`app.bsky.feed.getPostThread`](https://github.com/bluesky-social/atproto/blob/main/lexicons/app/bsky/feed/getPostThread.json)
- Hydrated post type: [`app.bsky.feed.defs#postView`](https://github.com/bluesky-social/atproto/blob/main/lexicons/app/bsky/feed/defs.json)
- Raw post record (what sits inside `record`): [`app.bsky.feed.post`](https://github.com/bluesky-social/atproto/blob/main/lexicons/app/bsky/feed/post.json)

The fields you can expect, at time of writing, are the fields on `postView`. Required fields are `uri`, `cid`, `author`, `record`, and `indexedAt`. Everything else can be missing.

Identity and the post itself:

- `uri`: AT-URI of the post
- `cid`: content hash of this version of the record
- `author`: a basic profile (`did`, `handle`, `displayName`, `avatar`, and so on). Same shape as `profileViewBasic` from Question 1, not the full detailed profile
- `record`: the raw `app.bsky.feed.post` record. Required fields on that record are `text` and `createdAt`. Optional fields are `facets` (mentions, links, hashtags in the text), `reply` (root and parent URIs if this is a reply), `embed` (images, video, link card, quoted post), `langs`, `labels` (self-labels / content warnings), and `tags`
- `embed`: the AppView's hydrated version of that embed (CDN image URLs, quoted-post preview, and so on)
- `indexedAt`: when the AppView last indexed this post

Counts and extras that the AppView computes (these are not stored as simple fields on the post record in the PDS):

- `likeCount`, `repostCount`, `replyCount`, `quoteCount`, `bookmarkCount`
- `labels`: moderation labels on the post
- `threadgate`: who is allowed to reply, if the author set a threadgate

Only present (or only meaningful) when you are authenticated:

- `viewer`: AT-URIs of *your* like or repost records if you have them, plus `bookmarked`, `threadMuted`, `replyDisabled`, `embeddingDisabled`, and `pinned`

What this API does *not* give you: the full list of likers, reposters, or every reply in a huge thread (thread depth is capped by `depth`). `getPosts` is the indexed post card, not a dump of the whole repo. For the raw post record as stored on the PDS (`app.bsky.feed.post`), see Method 2 below. That record has the text, timestamp, reply pointers, embeds, langs, tags, and self-labels. It does not have like/repost/reply/quote counts.

### Q2 Method 2: PDS backfill

A post in a repo is one record in the author's `app.bsky.feed.post` collection. The path in the CAR is `app.bsky.feed.post/<rkey>`. The AT-URI is `at://<author-did>/app.bsky.feed.post/<rkey>`. You can download the whole author repo with `getRepo`, list only posts with `com.atproto.repo.listRecords?collection=app.bsky.feed.post`, or fetch one post with `com.atproto.repo.getRecord` (same `repo` + `collection` + `rkey` split described in [Bluesky architecture](BLUESKY_ARCHITECTURE.md)).

The post record does not store engagement. A like of this post is a record in the *liker's* repo. A reply is a separate post in the *replier's* repo, with `reply.parent` / `reply.root` pointing at this URI. A quote is another post whose embed points at this URI. That is why a single `getRepo` cannot give you like counts, the like list, or the full thread.

If you get the data directly from the PDS, the following is available to you for a post:

- **`app.bsky.feed.post`**: the raw post. Required fields are `text` and `createdAt`. Optional fields, at time of writing, are `facets` (mentions, URLs, hashtags), `reply` (`root` and `parent` strong refs), `embed` (images, video, external link, quoted record, or record-plus-media), `langs`, `labels` (self-labels / content warnings), `tags`, and the deprecated `entities` field. Schema: [`app.bsky.feed.post`](https://github.com/bluesky-social/atproto/blob/main/lexicons/app/bsky/feed/post.json). Image and video bytes are blob refs on the embed. Fetch those with `getBlob`, not from the CAR.
- **`app.bsky.feed.threadgate`**: if the author limited who can reply. Same rkey as the root post, in the same repo. Fields: `post`, `allow` (mention / follower / following / list rules), `hiddenReplies`, `createdAt`.
- **`app.bsky.feed.postgate`**: if the author limited embedding or detached quote posts. Same rkey as the post. Fields: `post`, `embeddingRules`, `detachedEmbeddingUris`, `createdAt`.

In *this* author's repo you will also see **`app.bsky.feed.like`** and **`app.bsky.feed.repost`**, but those are likes and reposts *this author made of other posts*. Each is `{ subject: { uri, cid }, createdAt }`. They are not likes or reposts *of* the post you are studying.

What you will *not* find on the post record itself, even though `getPosts` returns them:

- `likeCount`, `repostCount`, `replyCount`, `quoteCount`, `bookmarkCount`
- the list of accounts that liked, reposted, replied, or quoted
- hydrated `author` profile and hydrated `embed` (CDN URLs, quoted-post preview)
- AppView `labels` from third-party labelers
- `viewer` fields (whether *you* liked it)
- bookmarks (private)

To reconstruct those from PDS data alone, you have to scan other repos (or use the firehose / an AppView). For most research questions that need counts or threads, the AppView API in Method 1 is the right tool. Use a PDS backfill when you need the raw authored records, a full user dump, or data the AppView does not keep.
