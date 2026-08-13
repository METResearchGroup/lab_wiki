# Bluesky Architecture Deep Dive

## Bluesky architecture

To learn more about Bluesky's architecture, see [this deep dive](https://docs.bsky.app/docs/advanced-guides/federation-architecture) from the Bluesky team. They do a great job of describing it in detail, though it's admittedly a bit engineering-heavy.

In simpler terms, there's 3 concepts to know:

- **PDS (Personal Data Server)**: these store your data (see the "How does Bluesky store your data?" section below for more info on how this works).
- **Relay**: this goes ...
- **AppView**: ...

## How does Bluesky store data?

### How does Facebook/Twitter/Google store your data?

When you have an account on Facebook/Twitter/Google, all your data is stored on their proprietary servers. This means that if you wanted to leave the platform and go somewhere else, you can't "download" your data and take it with you. These platforms own your data. They can train their AIs on it, they can block you at will, and they can monetize off your likeness.

### How does Bluesky store your data?

Bluesky designed a system that uses a [PDS (Personal Data Server)](https://github.com/bluesky-social/pds) to store all your data. Basically, a PDS is just a server, same as what Facebook/Twitter/Google would use, but the upside is that anyone can host their own server and plug it into the Bluesky ecosystem. Most people don't want to run their own PDS, so Bluesky has a few servers that they run themselves, but anyone could easily get the data that Bluesky has on ...

The Bluesky team has wider ambitions of building a decentralized social media network, defined by what they call the [AT Protocol](https://atproto.com/guides/overview), of which Bluesky is just the first user-facing product and the PDS is one component in this architecture.
