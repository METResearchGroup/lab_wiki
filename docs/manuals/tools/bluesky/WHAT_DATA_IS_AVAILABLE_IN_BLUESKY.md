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

## What can we get from getRepo vs. AppView

`getRepo` reads the current public records that one account wrote, from that account's PDS. The AppView is the indexed app layer on top of those repos. It adds counts, follower lists, thread trees, and hydrated profiles. The lists below are only the calls you need for info about one user or one post. Links match the official [HTTP API reference](https://docs.bsky.app/docs/category/http-reference), as of August 2026. For the field-level detail, see Question 1 and Question 2 above.

### What can we get from getRepo?

`com.atproto.sync.getRepo` returns a CAR file with every current public record in that repo. You can also pull one collection or one record instead of the whole dump. None of these calls give you follower lists, like lists on a post, or AppView counts. Those live in other people's repos, or on the AppView.

For a user (the records this account authored):

- `com.atproto.sync.getRepo`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-get-repo)
- `com.atproto.repo.describeRepo`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-describe-repo)
- `com.atproto.repo.listRecords`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-list-records)
- `com.atproto.repo.getRecord`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-get-record)
- `com.atproto.identity.resolveHandle`. [Link](https://docs.bsky.app/docs/api/com-atproto-identity-resolve-handle)
- `com.atproto.sync.listBlobs`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-list-blobs)
- `com.atproto.sync.getBlob`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-get-blob)

From those, the user-related collections you actually parse out are `app.bsky.actor.profile` (bio, display name, avatar and banner blob refs, pinned post), `app.bsky.graph.follow` (who they follow, not who follows them), and this account's own `app.bsky.feed.post`, `app.bsky.feed.like`, and `app.bsky.feed.repost` records. `resolveHandle` is only there if you start from a handle and need the DID. Blobs are avatars, banners, and post images. They are not inside the CAR.

For a post:

- `com.atproto.sync.getRepo`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-get-repo)
- `com.atproto.repo.listRecords`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-list-records)
- `com.atproto.repo.getRecord`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-get-record)
- `com.atproto.sync.getBlob`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-get-blob)

From those, the post-related records in the *author's* repo are `app.bsky.feed.post` (text, facets, reply pointers, embed blob refs, langs, tags, self-labels), plus `app.bsky.feed.threadgate` and `app.bsky.feed.postgate` if the author set them. Likes and reposts *of* that post are not in this repo.

### What can we get from AppView?

AppView calls return hydrated objects. Profiles include handle, CDN image URLs, and follower/follow/post counts. Posts include the raw record plus like/repost/reply/quote counts, a hydrated author, a hydrated embed, and (for `getPostThread`) parents and replies. Auth is optional on these. A session fills in extra `viewer` fields.

For a user:

- `app.bsky.actor.getProfile`. [Link](https://docs.bsky.app/docs/api/app-bsky-actor-get-profile)
- `app.bsky.actor.getProfiles`. [Link](https://docs.bsky.app/docs/api/app-bsky-actor-get-profiles)
- `app.bsky.graph.getFollows`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-follows)
- `app.bsky.graph.getFollowers`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-followers)
- `app.bsky.feed.getAuthorFeed`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-author-feed)

For a post:

- `app.bsky.feed.getPosts`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-posts)
- `app.bsky.feed.getPostThread`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-post-thread)
- `app.bsky.feed.getLikes`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-likes)
- `app.bsky.feed.getRepostedBy`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-reposted-by)
- `app.bsky.feed.getQuotes`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-quotes)

## Which requests require authentication and which do not?

This list is from the official [HTTP API reference](https://docs.bsky.app/docs/category/http-reference), the AT Protocol lexicons, and Bluesky's OpenAPI, as of August 2026. Record types and schema-only defs are omitted. Only query and procedure endpoints are listed.

A few public reads add extra fields if you send a session (for example `getProfile` and `getPostThread`). Those still work without auth, so they are in the second list. `createSession` and `createAccount` do not need a prior token. They are the login and signup calls, so they are also in the second list. Admin (`com.atproto.admin.*`) and Ozone (`tools.ozone.*`) need moderator or admin credentials, not a normal user session. DMs (`chat.bsky.*`) need a session, usually proxied to `api.bsky.chat`. Some unspecced and Jetstream archive endpoints are newer than the published docs cards. The link format still matches the HTTP reference slug.

### Requires authentication

- `app.bsky.actor.getPreferences`. [Link](https://docs.bsky.app/docs/api/app-bsky-actor-get-preferences)
- `app.bsky.actor.putPreferences`. [Link](https://docs.bsky.app/docs/api/app-bsky-actor-put-preferences)
- `app.bsky.ageassurance.begin`. [Link](https://docs.bsky.app/docs/api/app-bsky-ageassurance-begin)
- `app.bsky.bookmark.createBookmark`. [Link](https://docs.bsky.app/docs/api/app-bsky-bookmark-create-bookmark)
- `app.bsky.bookmark.deleteBookmark`. [Link](https://docs.bsky.app/docs/api/app-bsky-bookmark-delete-bookmark)
- `app.bsky.bookmark.getBookmarks`. [Link](https://docs.bsky.app/docs/api/app-bsky-bookmark-get-bookmarks)
- `app.bsky.contact.dismissMatch`. [Link](https://docs.bsky.app/docs/api/app-bsky-contact-dismiss-match)
- `app.bsky.contact.getMatches`. [Link](https://docs.bsky.app/docs/api/app-bsky-contact-get-matches)
- `app.bsky.contact.getSyncStatus`. [Link](https://docs.bsky.app/docs/api/app-bsky-contact-get-sync-status)
- `app.bsky.contact.importContacts`. [Link](https://docs.bsky.app/docs/api/app-bsky-contact-import-contacts)
- `app.bsky.contact.removeData`. [Link](https://docs.bsky.app/docs/api/app-bsky-contact-remove-data)
- `app.bsky.contact.sendNotification`. [Link](https://docs.bsky.app/docs/api/app-bsky-contact-send-notification)
- `app.bsky.contact.startPhoneVerification`. [Link](https://docs.bsky.app/docs/api/app-bsky-contact-start-phone-verification)
- `app.bsky.contact.verifyPhone`. [Link](https://docs.bsky.app/docs/api/app-bsky-contact-verify-phone)
- `app.bsky.draft.createDraft`. [Link](https://docs.bsky.app/docs/api/app-bsky-draft-create-draft)
- `app.bsky.draft.deleteDraft`. [Link](https://docs.bsky.app/docs/api/app-bsky-draft-delete-draft)
- `app.bsky.draft.getDrafts`. [Link](https://docs.bsky.app/docs/api/app-bsky-draft-get-drafts)
- `app.bsky.draft.updateDraft`. [Link](https://docs.bsky.app/docs/api/app-bsky-draft-update-draft)
- `app.bsky.feed.getActorLikes`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-actor-likes)
- `app.bsky.feed.getSuggestedFeeds`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-suggested-feeds)
- `app.bsky.feed.getTimeline`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-timeline)
- `app.bsky.feed.sendInteractions`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-send-interactions)
- `app.bsky.graph.getBlocks`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-blocks)
- `app.bsky.graph.getListBlocks`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-list-blocks)
- `app.bsky.graph.getListMutes`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-list-mutes)
- `app.bsky.graph.getListsWithMembership`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-lists-with-membership)
- `app.bsky.graph.getMutes`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-mutes)
- `app.bsky.graph.getStarterPacksWithMembership`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-starter-packs-with-membership)
- `app.bsky.graph.muteActor`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-mute-actor)
- `app.bsky.graph.muteActorList`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-mute-actor-list)
- `app.bsky.graph.muteThread`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-mute-thread)
- `app.bsky.graph.unmuteActor`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-unmute-actor)
- `app.bsky.graph.unmuteActorList`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-unmute-actor-list)
- `app.bsky.graph.unmuteThread`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-unmute-thread)
- `app.bsky.notification.getPreferences`. [Link](https://docs.bsky.app/docs/api/app-bsky-notification-get-preferences)
- `app.bsky.notification.getUnreadCount`. [Link](https://docs.bsky.app/docs/api/app-bsky-notification-get-unread-count)
- `app.bsky.notification.listActivitySubscriptions`. [Link](https://docs.bsky.app/docs/api/app-bsky-notification-list-activity-subscriptions)
- `app.bsky.notification.listNotifications`. [Link](https://docs.bsky.app/docs/api/app-bsky-notification-list-notifications)
- `app.bsky.notification.putActivitySubscription`. [Link](https://docs.bsky.app/docs/api/app-bsky-notification-put-activity-subscription)
- `app.bsky.notification.putPreferences`. [Link](https://docs.bsky.app/docs/api/app-bsky-notification-put-preferences)
- `app.bsky.notification.putPreferencesV2`. [Link](https://docs.bsky.app/docs/api/app-bsky-notification-put-preferences-v2)
- `app.bsky.notification.registerPush`. [Link](https://docs.bsky.app/docs/api/app-bsky-notification-register-push)
- `app.bsky.notification.unregisterPush`. [Link](https://docs.bsky.app/docs/api/app-bsky-notification-unregister-push)
- `app.bsky.notification.updateSeen`. [Link](https://docs.bsky.app/docs/api/app-bsky-notification-update-seen)
- `app.bsky.unspecced.getAgeAssuranceState`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-age-assurance-state)
- `app.bsky.unspecced.initAgeAssurance`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-init-age-assurance)
- `app.bsky.video.abortUpload`. [Link](https://docs.bsky.app/docs/api/app-bsky-video-abort-upload)
- `app.bsky.video.finishUpload`. [Link](https://docs.bsky.app/docs/api/app-bsky-video-finish-upload)
- `app.bsky.video.getUploadLimits`. [Link](https://docs.bsky.app/docs/api/app-bsky-video-get-upload-limits)
- `app.bsky.video.getUploadStatus`. [Link](https://docs.bsky.app/docs/api/app-bsky-video-get-upload-status)
- `app.bsky.video.startUpload`. [Link](https://docs.bsky.app/docs/api/app-bsky-video-start-upload)
- `app.bsky.video.uploadPart`. [Link](https://docs.bsky.app/docs/api/app-bsky-video-upload-part)
- `app.bsky.video.uploadVideo`. [Link](https://docs.bsky.app/docs/api/app-bsky-video-upload-video)
- `chat.bsky.actor.deleteAccount`. [Link](https://docs.bsky.app/docs/api/chat-bsky-actor-delete-account)
- `chat.bsky.actor.exportAccountData`. [Link](https://docs.bsky.app/docs/api/chat-bsky-actor-export-account-data)
- `chat.bsky.actor.getStatus`. [Link](https://docs.bsky.app/docs/api/chat-bsky-actor-get-status)
- `chat.bsky.convo.acceptConvo`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-accept-convo)
- `chat.bsky.convo.addReaction`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-add-reaction)
- `chat.bsky.convo.deleteMessageForSelf`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-delete-message-for-self)
- `chat.bsky.convo.getConvo`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-get-convo)
- `chat.bsky.convo.getConvoAvailability`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-get-convo-availability)
- `chat.bsky.convo.getConvoForMembers`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-get-convo-for-members)
- `chat.bsky.convo.getConvoMembers`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-get-convo-members)
- `chat.bsky.convo.getLog`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-get-log)
- `chat.bsky.convo.getMessages`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-get-messages)
- `chat.bsky.convo.getUnreadCounts`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-get-unread-counts)
- `chat.bsky.convo.leaveConvo`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-leave-convo)
- `chat.bsky.convo.listConvoRequests`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-list-convo-requests)
- `chat.bsky.convo.listConvos`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-list-convos)
- `chat.bsky.convo.lockConvo`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-lock-convo)
- `chat.bsky.convo.muteConvo`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-mute-convo)
- `chat.bsky.convo.removeReaction`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-remove-reaction)
- `chat.bsky.convo.sendMessage`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-send-message)
- `chat.bsky.convo.sendMessageBatch`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-send-message-batch)
- `chat.bsky.convo.unlockConvo`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-unlock-convo)
- `chat.bsky.convo.unmuteConvo`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-unmute-convo)
- `chat.bsky.convo.updateAllRead`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-update-all-read)
- `chat.bsky.convo.updateRead`. [Link](https://docs.bsky.app/docs/api/chat-bsky-convo-update-read)
- `chat.bsky.group.addMembers`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-add-members)
- `chat.bsky.group.approveJoinRequest`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-approve-join-request)
- `chat.bsky.group.createGroup`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-create-group)
- `chat.bsky.group.createJoinLink`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-create-join-link)
- `chat.bsky.group.disableJoinLink`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-disable-join-link)
- `chat.bsky.group.editGroup`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-edit-group)
- `chat.bsky.group.editJoinLink`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-edit-join-link)
- `chat.bsky.group.enableJoinLink`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-enable-join-link)
- `chat.bsky.group.getGroupPublicInfo`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-get-group-public-info)
- `chat.bsky.group.listJoinRequests`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-list-join-requests)
- `chat.bsky.group.listMutualGroups`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-list-mutual-groups)
- `chat.bsky.group.rejectJoinRequest`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-reject-join-request)
- `chat.bsky.group.removeMembers`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-remove-members)
- `chat.bsky.group.requestJoin`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-request-join)
- `chat.bsky.group.updateJoinRequestsRead`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-update-join-requests-read)
- `chat.bsky.group.withdrawJoinRequest`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-withdraw-join-request)
- `chat.bsky.moderation.getActorMetadata`. [Link](https://docs.bsky.app/docs/api/chat-bsky-moderation-get-actor-metadata)
- `chat.bsky.moderation.getConvo`. [Link](https://docs.bsky.app/docs/api/chat-bsky-moderation-get-convo)
- `chat.bsky.moderation.getConvoMembers`. [Link](https://docs.bsky.app/docs/api/chat-bsky-moderation-get-convo-members)
- `chat.bsky.moderation.getConvos`. [Link](https://docs.bsky.app/docs/api/chat-bsky-moderation-get-convos)
- `chat.bsky.moderation.getMessageContext`. [Link](https://docs.bsky.app/docs/api/chat-bsky-moderation-get-message-context)
- `chat.bsky.moderation.updateActorAccess`. [Link](https://docs.bsky.app/docs/api/chat-bsky-moderation-update-actor-access)
- `chat.bsky.notification.getPreferences`. [Link](https://docs.bsky.app/docs/api/chat-bsky-notification-get-preferences)
- `chat.bsky.notification.putPreferences`. [Link](https://docs.bsky.app/docs/api/chat-bsky-notification-put-preferences)
- `com.atproto.admin.deleteAccount`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-delete-account)
- `com.atproto.admin.disableAccountInvites`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-disable-account-invites)
- `com.atproto.admin.disableInviteCodes`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-disable-invite-codes)
- `com.atproto.admin.enableAccountInvites`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-enable-account-invites)
- `com.atproto.admin.getAccountInfo`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-get-account-info)
- `com.atproto.admin.getAccountInfos`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-get-account-infos)
- `com.atproto.admin.getInviteCodes`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-get-invite-codes)
- `com.atproto.admin.getSubjectStatus`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-get-subject-status)
- `com.atproto.admin.searchAccounts`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-search-accounts)
- `com.atproto.admin.sendEmail`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-send-email)
- `com.atproto.admin.updateAccountEmail`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-update-account-email)
- `com.atproto.admin.updateAccountHandle`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-update-account-handle)
- `com.atproto.admin.updateAccountPassword`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-update-account-password)
- `com.atproto.admin.updateAccountSigningKey`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-update-account-signing-key)
- `com.atproto.admin.updateSubjectStatus`. [Link](https://docs.bsky.app/docs/api/com-atproto-admin-update-subject-status)
- `com.atproto.identity.getRecommendedDidCredentials`. [Link](https://docs.bsky.app/docs/api/com-atproto-identity-get-recommended-did-credentials)
- `com.atproto.identity.requestPlcOperationSignature`. [Link](https://docs.bsky.app/docs/api/com-atproto-identity-request-plc-operation-signature)
- `com.atproto.identity.signPlcOperation`. [Link](https://docs.bsky.app/docs/api/com-atproto-identity-sign-plc-operation)
- `com.atproto.identity.submitPlcOperation`. [Link](https://docs.bsky.app/docs/api/com-atproto-identity-submit-plc-operation)
- `com.atproto.identity.updateHandle`. [Link](https://docs.bsky.app/docs/api/com-atproto-identity-update-handle)
- `com.atproto.moderation.createReport`. [Link](https://docs.bsky.app/docs/api/com-atproto-moderation-create-report)
- `com.atproto.repo.applyWrites`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-apply-writes)
- `com.atproto.repo.createRecord`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-create-record)
- `com.atproto.repo.deleteRecord`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-delete-record)
- `com.atproto.repo.importRepo`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-import-repo)
- `com.atproto.repo.listMissingBlobs`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-list-missing-blobs)
- `com.atproto.repo.putRecord`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-put-record)
- `com.atproto.repo.uploadBlob`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-upload-blob)
- `com.atproto.server.activateAccount`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-activate-account)
- `com.atproto.server.checkAccountStatus`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-check-account-status)
- `com.atproto.server.confirmEmail`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-confirm-email)
- `com.atproto.server.createAppPassword`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-create-app-password)
- `com.atproto.server.createInviteCode`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-create-invite-code)
- `com.atproto.server.createInviteCodes`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-create-invite-codes)
- `com.atproto.server.deactivateAccount`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-deactivate-account)
- `com.atproto.server.deleteAccount`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-delete-account)
- `com.atproto.server.deleteSession`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-delete-session)
- `com.atproto.server.getAccountInviteCodes`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-get-account-invite-codes)
- `com.atproto.server.getServiceAuth`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-get-service-auth)
- `com.atproto.server.getSession`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-get-session)
- `com.atproto.server.listAppPasswords`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-list-app-passwords)
- `com.atproto.server.refreshSession`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-refresh-session)
- `com.atproto.server.requestAccountDelete`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-request-account-delete)
- `com.atproto.server.requestEmailConfirmation`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-request-email-confirmation)
- `com.atproto.server.requestEmailUpdate`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-request-email-update)
- `com.atproto.server.revokeAppPassword`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-revoke-app-password)
- `com.atproto.server.updateEmail`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-update-email)
- `com.atproto.temp.addReservedHandle`. [Link](https://docs.bsky.app/docs/api/com-atproto-temp-add-reserved-handle)
- `com.atproto.temp.checkSignupQueue`. [Link](https://docs.bsky.app/docs/api/com-atproto-temp-check-signup-queue)
- `com.atproto.temp.revokeAccountCredentials`. [Link](https://docs.bsky.app/docs/api/com-atproto-temp-revoke-account-credentials)
- `network.bsky.jetstream.getBlock`. [Link](https://docs.bsky.app/docs/api/network-bsky-jetstream-get-block)
- `network.bsky.jetstream.getImportStatus`. [Link](https://docs.bsky.app/docs/api/network-bsky-jetstream-get-import-status)
- `network.bsky.jetstream.getSegment`. [Link](https://docs.bsky.app/docs/api/network-bsky-jetstream-get-segment)
- `network.bsky.jetstream.importTimestamps`. [Link](https://docs.bsky.app/docs/api/network-bsky-jetstream-import-timestamps)
- `network.bsky.jetstream.listSegments`. [Link](https://docs.bsky.app/docs/api/network-bsky-jetstream-list-segments)
- `network.bsky.jetstream.planSnapshot`. [Link](https://docs.bsky.app/docs/api/network-bsky-jetstream-plan-snapshot)
- `tools.ozone.communication.createTemplate`. [Link](https://docs.bsky.app/docs/api/tools-ozone-communication-create-template)
- `tools.ozone.communication.deleteTemplate`. [Link](https://docs.bsky.app/docs/api/tools-ozone-communication-delete-template)
- `tools.ozone.communication.listTemplates`. [Link](https://docs.bsky.app/docs/api/tools-ozone-communication-list-templates)
- `tools.ozone.communication.updateTemplate`. [Link](https://docs.bsky.app/docs/api/tools-ozone-communication-update-template)
- `tools.ozone.hosting.getAccountHistory`. [Link](https://docs.bsky.app/docs/api/tools-ozone-hosting-get-account-history)
- `tools.ozone.moderation.cancelScheduledActions`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-cancel-scheduled-actions)
- `tools.ozone.moderation.emitEvent`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-emit-event)
- `tools.ozone.moderation.getAccountTimeline`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-get-account-timeline)
- `tools.ozone.moderation.getEvent`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-get-event)
- `tools.ozone.moderation.getRecord`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-get-record)
- `tools.ozone.moderation.getRecords`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-get-records)
- `tools.ozone.moderation.getRepo`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-get-repo)
- `tools.ozone.moderation.getReporterStats`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-get-reporter-stats)
- `tools.ozone.moderation.getRepos`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-get-repos)
- `tools.ozone.moderation.getSubjects`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-get-subjects)
- `tools.ozone.moderation.listScheduledActions`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-list-scheduled-actions)
- `tools.ozone.moderation.queryEvents`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-query-events)
- `tools.ozone.moderation.queryStatuses`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-query-statuses)
- `tools.ozone.moderation.scheduleAction`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-schedule-action)
- `tools.ozone.moderation.searchRepos`. [Link](https://docs.bsky.app/docs/api/tools-ozone-moderation-search-repos)
- `tools.ozone.queue.assignModerator`. [Link](https://docs.bsky.app/docs/api/tools-ozone-queue-assign-moderator)
- `tools.ozone.queue.createQueue`. [Link](https://docs.bsky.app/docs/api/tools-ozone-queue-create-queue)
- `tools.ozone.queue.deleteQueue`. [Link](https://docs.bsky.app/docs/api/tools-ozone-queue-delete-queue)
- `tools.ozone.queue.getAssignments`. [Link](https://docs.bsky.app/docs/api/tools-ozone-queue-get-assignments)
- `tools.ozone.queue.listQueues`. [Link](https://docs.bsky.app/docs/api/tools-ozone-queue-list-queues)
- `tools.ozone.queue.routeReports`. [Link](https://docs.bsky.app/docs/api/tools-ozone-queue-route-reports)
- `tools.ozone.queue.unassignModerator`. [Link](https://docs.bsky.app/docs/api/tools-ozone-queue-unassign-moderator)
- `tools.ozone.queue.updateQueue`. [Link](https://docs.bsky.app/docs/api/tools-ozone-queue-update-queue)
- `tools.ozone.report.assignModerator`. [Link](https://docs.bsky.app/docs/api/tools-ozone-report-assign-moderator)
- `tools.ozone.report.closeReports`. [Link](https://docs.bsky.app/docs/api/tools-ozone-report-close-reports)
- `tools.ozone.report.createActivity`. [Link](https://docs.bsky.app/docs/api/tools-ozone-report-create-activity)
- `tools.ozone.report.getAssignments`. [Link](https://docs.bsky.app/docs/api/tools-ozone-report-get-assignments)
- `tools.ozone.report.getHistoricalStats`. [Link](https://docs.bsky.app/docs/api/tools-ozone-report-get-historical-stats)
- `tools.ozone.report.getLatestReport`. [Link](https://docs.bsky.app/docs/api/tools-ozone-report-get-latest-report)
- `tools.ozone.report.getLiveStats`. [Link](https://docs.bsky.app/docs/api/tools-ozone-report-get-live-stats)
- `tools.ozone.report.getReport`. [Link](https://docs.bsky.app/docs/api/tools-ozone-report-get-report)
- `tools.ozone.report.listActivities`. [Link](https://docs.bsky.app/docs/api/tools-ozone-report-list-activities)
- `tools.ozone.report.queryActivities`. [Link](https://docs.bsky.app/docs/api/tools-ozone-report-query-activities)
- `tools.ozone.report.queryReports`. [Link](https://docs.bsky.app/docs/api/tools-ozone-report-query-reports)
- `tools.ozone.report.reassignQueue`. [Link](https://docs.bsky.app/docs/api/tools-ozone-report-reassign-queue)
- `tools.ozone.report.refreshStats`. [Link](https://docs.bsky.app/docs/api/tools-ozone-report-refresh-stats)
- `tools.ozone.report.unassignModerator`. [Link](https://docs.bsky.app/docs/api/tools-ozone-report-unassign-moderator)
- `tools.ozone.safelink.addRule`. [Link](https://docs.bsky.app/docs/api/tools-ozone-safelink-add-rule)
- `tools.ozone.safelink.queryEvents`. [Link](https://docs.bsky.app/docs/api/tools-ozone-safelink-query-events)
- `tools.ozone.safelink.queryRules`. [Link](https://docs.bsky.app/docs/api/tools-ozone-safelink-query-rules)
- `tools.ozone.safelink.removeRule`. [Link](https://docs.bsky.app/docs/api/tools-ozone-safelink-remove-rule)
- `tools.ozone.safelink.updateRule`. [Link](https://docs.bsky.app/docs/api/tools-ozone-safelink-update-rule)
- `tools.ozone.server.getConfig`. [Link](https://docs.bsky.app/docs/api/tools-ozone-server-get-config)
- `tools.ozone.set.addValues`. [Link](https://docs.bsky.app/docs/api/tools-ozone-set-add-values)
- `tools.ozone.set.deleteSet`. [Link](https://docs.bsky.app/docs/api/tools-ozone-set-delete-set)
- `tools.ozone.set.deleteValues`. [Link](https://docs.bsky.app/docs/api/tools-ozone-set-delete-values)
- `tools.ozone.set.getValues`. [Link](https://docs.bsky.app/docs/api/tools-ozone-set-get-values)
- `tools.ozone.set.querySets`. [Link](https://docs.bsky.app/docs/api/tools-ozone-set-query-sets)
- `tools.ozone.set.upsertSet`. [Link](https://docs.bsky.app/docs/api/tools-ozone-set-upsert-set)
- `tools.ozone.setting.listOptions`. [Link](https://docs.bsky.app/docs/api/tools-ozone-setting-list-options)
- `tools.ozone.setting.removeOptions`. [Link](https://docs.bsky.app/docs/api/tools-ozone-setting-remove-options)
- `tools.ozone.setting.upsertOption`. [Link](https://docs.bsky.app/docs/api/tools-ozone-setting-upsert-option)
- `tools.ozone.signature.findCorrelation`. [Link](https://docs.bsky.app/docs/api/tools-ozone-signature-find-correlation)
- `tools.ozone.signature.findRelatedAccounts`. [Link](https://docs.bsky.app/docs/api/tools-ozone-signature-find-related-accounts)
- `tools.ozone.signature.searchAccounts`. [Link](https://docs.bsky.app/docs/api/tools-ozone-signature-search-accounts)
- `tools.ozone.team.addMember`. [Link](https://docs.bsky.app/docs/api/tools-ozone-team-add-member)
- `tools.ozone.team.deleteMember`. [Link](https://docs.bsky.app/docs/api/tools-ozone-team-delete-member)
- `tools.ozone.team.listMembers`. [Link](https://docs.bsky.app/docs/api/tools-ozone-team-list-members)
- `tools.ozone.team.updateMember`. [Link](https://docs.bsky.app/docs/api/tools-ozone-team-update-member)
- `tools.ozone.verification.grantVerifications`. [Link](https://docs.bsky.app/docs/api/tools-ozone-verification-grant-verifications)
- `tools.ozone.verification.listVerifications`. [Link](https://docs.bsky.app/docs/api/tools-ozone-verification-list-verifications)
- `tools.ozone.verification.revokeVerifications`. [Link](https://docs.bsky.app/docs/api/tools-ozone-verification-revoke-verifications)

### Doesn't require authentication

- `app.bsky.actor.getProfile`. [Link](https://docs.bsky.app/docs/api/app-bsky-actor-get-profile)
- `app.bsky.actor.getProfiles`. [Link](https://docs.bsky.app/docs/api/app-bsky-actor-get-profiles)
- `app.bsky.actor.getSuggestions`. [Link](https://docs.bsky.app/docs/api/app-bsky-actor-get-suggestions)
- `app.bsky.actor.searchActors`. [Link](https://docs.bsky.app/docs/api/app-bsky-actor-search-actors)
- `app.bsky.actor.searchActorsTypeahead`. [Link](https://docs.bsky.app/docs/api/app-bsky-actor-search-actors-typeahead)
- `app.bsky.ageassurance.getConfig`. [Link](https://docs.bsky.app/docs/api/app-bsky-ageassurance-get-config)
- `app.bsky.ageassurance.getState`. [Link](https://docs.bsky.app/docs/api/app-bsky-ageassurance-get-state)
- `app.bsky.embed.getEmbedExternalView`. [Link](https://docs.bsky.app/docs/api/app-bsky-embed-get-embed-external-view)
- `app.bsky.feed.describeFeedGenerator`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-describe-feed-generator)
- `app.bsky.feed.getActorFeeds`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-actor-feeds)
- `app.bsky.feed.getAuthorFeed`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-author-feed)
- `app.bsky.feed.getFeed`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-feed)
- `app.bsky.feed.getFeedGenerator`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-feed-generator)
- `app.bsky.feed.getFeedGenerators`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-feed-generators)
- `app.bsky.feed.getFeedSkeleton`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-feed-skeleton)
- `app.bsky.feed.getLikes`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-likes)
- `app.bsky.feed.getListFeed`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-list-feed)
- `app.bsky.feed.getPostThread`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-post-thread)
- `app.bsky.feed.getPosts`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-posts)
- `app.bsky.feed.getQuotes`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-quotes)
- `app.bsky.feed.getRepostedBy`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-get-reposted-by)
- `app.bsky.feed.searchPosts`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-search-posts)
- `app.bsky.feed.searchPostsV2`. [Link](https://docs.bsky.app/docs/api/app-bsky-feed-search-posts-v2)
- `app.bsky.graph.getActorStarterPacks`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-actor-starter-packs)
- `app.bsky.graph.getFollowers`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-followers)
- `app.bsky.graph.getFollows`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-follows)
- `app.bsky.graph.getKnownFollowers`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-known-followers)
- `app.bsky.graph.getList`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-list)
- `app.bsky.graph.getLists`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-lists)
- `app.bsky.graph.getRelationships`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-relationships)
- `app.bsky.graph.getStarterPack`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-starter-pack)
- `app.bsky.graph.getStarterPacks`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-starter-packs)
- `app.bsky.graph.getSuggestedFollowsByActor`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-get-suggested-follows-by-actor)
- `app.bsky.graph.searchStarterPacks`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-search-starter-packs)
- `app.bsky.graph.searchStarterPacksV2`. [Link](https://docs.bsky.app/docs/api/app-bsky-graph-search-starter-packs-v2)
- `app.bsky.labeler.getServices`. [Link](https://docs.bsky.app/docs/api/app-bsky-labeler-get-services)
- `app.bsky.unspecced.getConfig`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-config)
- `app.bsky.unspecced.getOnboardingSuggestedStarterPacks`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-onboarding-suggested-starter-packs)
- `app.bsky.unspecced.getOnboardingSuggestedStarterPacksSkeleton`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-onboarding-suggested-starter-packs-skeleton)
- `app.bsky.unspecced.getOnboardingSuggestedUsersSkeleton`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-onboarding-suggested-users-skeleton)
- `app.bsky.unspecced.getPopularFeedGenerators`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-popular-feed-generators)
- `app.bsky.unspecced.getPostThreadOtherV2`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-post-thread-other-v2)
- `app.bsky.unspecced.getPostThreadV2`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-post-thread-v2)
- `app.bsky.unspecced.getSuggestedFeeds`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-suggested-feeds)
- `app.bsky.unspecced.getSuggestedFeedsSkeleton`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-suggested-feeds-skeleton)
- `app.bsky.unspecced.getSuggestedOnboardingUsers`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-suggested-onboarding-users)
- `app.bsky.unspecced.getSuggestedStarterPacks`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-suggested-starter-packs)
- `app.bsky.unspecced.getSuggestedStarterPacksSkeleton`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-suggested-starter-packs-skeleton)
- `app.bsky.unspecced.getSuggestedUsers`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-suggested-users)
- `app.bsky.unspecced.getSuggestedUsersForDiscover`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-suggested-users-for-discover)
- `app.bsky.unspecced.getSuggestedUsersForDiscoverSkeleton`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-suggested-users-for-discover-skeleton)
- `app.bsky.unspecced.getSuggestedUsersForExplore`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-suggested-users-for-explore)
- `app.bsky.unspecced.getSuggestedUsersForExploreSkeleton`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-suggested-users-for-explore-skeleton)
- `app.bsky.unspecced.getSuggestedUsersForSeeMore`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-suggested-users-for-see-more)
- `app.bsky.unspecced.getSuggestedUsersForSeeMoreSkeleton`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-suggested-users-for-see-more-skeleton)
- `app.bsky.unspecced.getSuggestedUsersSkeleton`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-suggested-users-skeleton)
- `app.bsky.unspecced.getSuggestionsSkeleton`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-suggestions-skeleton)
- `app.bsky.unspecced.getTaggedSuggestions`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-tagged-suggestions)
- `app.bsky.unspecced.getTrendingTopics`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-trending-topics)
- `app.bsky.unspecced.getTrends`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-trends)
- `app.bsky.unspecced.getTrendsSkeleton`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-get-trends-skeleton)
- `app.bsky.unspecced.searchActorsSkeleton`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-search-actors-skeleton)
- `app.bsky.unspecced.searchPostsSkeleton`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-search-posts-skeleton)
- `app.bsky.unspecced.searchStarterPacksSkeleton`. [Link](https://docs.bsky.app/docs/api/app-bsky-unspecced-search-starter-packs-skeleton)
- `app.bsky.video.getJobStatus`. [Link](https://docs.bsky.app/docs/api/app-bsky-video-get-job-status)
- `chat.bsky.group.getJoinLinkPreviews`. [Link](https://docs.bsky.app/docs/api/chat-bsky-group-get-join-link-previews)
- `com.atproto.identity.refreshIdentity`. [Link](https://docs.bsky.app/docs/api/com-atproto-identity-refresh-identity)
- `com.atproto.identity.resolveDid`. [Link](https://docs.bsky.app/docs/api/com-atproto-identity-resolve-did)
- `com.atproto.identity.resolveHandle`. [Link](https://docs.bsky.app/docs/api/com-atproto-identity-resolve-handle)
- `com.atproto.identity.resolveIdentity`. [Link](https://docs.bsky.app/docs/api/com-atproto-identity-resolve-identity)
- `com.atproto.label.queryLabels`. [Link](https://docs.bsky.app/docs/api/com-atproto-label-query-labels)
- `com.atproto.label.subscribeLabels`. [Link](https://docs.bsky.app/docs/api/com-atproto-label-subscribe-labels)
- `com.atproto.lexicon.resolveLexicon`. [Link](https://docs.bsky.app/docs/api/com-atproto-lexicon-resolve-lexicon)
- `com.atproto.repo.describeRepo`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-describe-repo)
- `com.atproto.repo.getRecord`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-get-record)
- `com.atproto.repo.listRecords`. [Link](https://docs.bsky.app/docs/api/com-atproto-repo-list-records)
- `com.atproto.server.createAccount`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-create-account)
- `com.atproto.server.createSession`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-create-session)
- `com.atproto.server.describeServer`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-describe-server)
- `com.atproto.server.requestPasswordReset`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-request-password-reset)
- `com.atproto.server.reserveSigningKey`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-reserve-signing-key)
- `com.atproto.server.resetPassword`. [Link](https://docs.bsky.app/docs/api/com-atproto-server-reset-password)
- `com.atproto.sync.getBlob`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-get-blob)
- `com.atproto.sync.getBlocks`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-get-blocks)
- `com.atproto.sync.getCheckout`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-get-checkout)
- `com.atproto.sync.getHead`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-get-head)
- `com.atproto.sync.getHostStatus`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-get-host-status)
- `com.atproto.sync.getLatestCommit`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-get-latest-commit)
- `com.atproto.sync.getRecord`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-get-record)
- `com.atproto.sync.getRepo`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-get-repo)
- `com.atproto.sync.getRepoStatus`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-get-repo-status)
- `com.atproto.sync.listBlobs`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-list-blobs)
- `com.atproto.sync.listHosts`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-list-hosts)
- `com.atproto.sync.listRepos`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-list-repos)
- `com.atproto.sync.listReposByCollection`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-list-repos-by-collection)
- `com.atproto.sync.notifyOfUpdate`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-notify-of-update)
- `com.atproto.sync.requestCrawl`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-request-crawl)
- `com.atproto.sync.subscribeRepos`. [Link](https://docs.bsky.app/docs/api/com-atproto-sync-subscribe-repos)
- `com.atproto.temp.checkHandleAvailability`. [Link](https://docs.bsky.app/docs/api/com-atproto-temp-check-handle-availability)
- `com.atproto.temp.dereferenceScope`. [Link](https://docs.bsky.app/docs/api/com-atproto-temp-dereference-scope)
- `com.atproto.temp.fetchLabels`. [Link](https://docs.bsky.app/docs/api/com-atproto-temp-fetch-labels)
- `com.atproto.temp.requestPhoneVerification`. [Link](https://docs.bsky.app/docs/api/com-atproto-temp-request-phone-verification)
- `network.bsky.jetstream.getZstdDictionary`. [Link](https://docs.bsky.app/docs/api/network-bsky-jetstream-get-zstd-dictionary)
