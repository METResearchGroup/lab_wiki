# Bluesky Architecture Deep Dive

## Bluesky architecture

To learn more about Bluesky's architecture, see [this deep dive](https://docs.bsky.app/docs/advanced-guides/federation-architecture) from the Bluesky team. They do a great job of describing it in detail, though it's admittedly a bit engineering-heavy. They also do a deeper dive explanation of [each component individually and how they fit together](https://atproto.com/guides/the-at-stack).

In simpler terms, there's 3 concepts to know:

- **PDS (Personal Data Server)**: these store your data (see the "How does Bluesky store your data?" section below for more info on how this works).
- **Relay**: this is a service that, every so often, scrapes the data across all the PDSes, cleans it up (deduplicates, verifies, etc.) and then makes the results available to anything that's using it. Here's a full writeup of [the Bluesky Relay](https://bsky.network/docs/relay/) design. The main Bluesky relay powers a few downstream services, such as [Bluesky's firehose](https://bsky.network/docs/consuming-the-firehose/), the [Bluesky Jetstream](https://bsky.network/docs/jetstream/), and AppViews (see below). ...
- **AppView**: ...

## How does Bluesky store data?

### How does Facebook/Twitter/Google store your data?

When you have an account on Facebook/Twitter/Google, all your data is stored on their proprietary servers. This means that if you wanted to leave the platform and go somewhere else, you can't "download" your data and take it with you. These platforms own your data. They can train their AIs on it, they can block you at will, and they can monetize off your likeness.

### How does Bluesky store your data?

Bluesky designed a system that uses a [PDS (Personal Data Server)](https://github.com/bluesky-social/pds) to store all your data. Basically, a PDS is just a server, same as what Facebook/Twitter/Google would use, but the upside is that anyone can host their own server and plug it into the Bluesky ecosystem. Most people don't want to run their own PDS, so Bluesky has a few servers that they run themselves, but anyone could easily get the data that Bluesky has on ...

The Bluesky team has wider ambitions of building a decentralized social media network, defined by what they call the [AT Protocol](https://atproto.com/guides/overview), of which Bluesky is just the first user-facing product and the PDS is one component in this architecture.

#### How is the data stored?

An individual person's data is stored in something called a [repository](https://atproto.com/guides/data-repos) (aka `repo`).

...

At any time you can export the data that Bluesky has on you (including your posts, likes, follows, etc.), see [this guide](https://atproto.com/blog/repo-export) for more steps on that.

#### What happens when you get your data?

When you get your data, ...

#### Example: getting information about a post

A Bluesky post is identified by its [uri](https://atproto.com/specs/at-uri-scheme), which is basically its ID. A uri can look something like this: `at://did:plc:alice123/app.bsky.feed.post/3kxyz`.

Every uri has three key parts:

1. Repo: the "folder" of data.
2. Collection: the "record type". At a high level, this generally means "posts" or "profiles" or "likes". In a literal sense, it's anything that shares the same schema.
3. `rkey`: The record’s key, unique within that repository and collection. This is used for looking up a certain record, for if you need to read or update it. It also is used to reference a record within another record (for example, in a "reply" record, the "reply" record needs to reference the post that it is replying to).

#### Analogy: databases

For those familiar with relational databases, a record is unique on the combination of `PRIMARY KEY = (repo_did, collection, rkey)`.

For those familiar with document stores, the equivalent setup is something like this:

```markdown
repository: did:plc:alice
  collection: app.bsky.feed.post
    rkey: 3abc...  -> post document
    rkey: 3def...  -> post document

  collection: app.bsky.feed.like
    rkey: 3ghi...  -> like document
```

#### ...

In the example, the parts of the uri are:

- repo = did:plc:alice123
- collection = app.bsky.feed.post
- rkey = 3kxyz

##### What's happening under the hood during retrieval?

When you read Bluesky's docs, you'll see all these references to something called a "Merkle tree". This is basically a data structure that enables very efficient lookups as well as data updates. There's a few more details to it that are outside the scope of this work.

But basically, what happens is once a PDS receives 
