# How do we get Bluesky data?

Here are a series of 4 approaches for how to get data from Bluesky. This is in order of easiest to most difficult, I'd suggest trying them in this order.

## Approach 1: Consult existing lab Bluesky datasets

We've invested quite a bit of work on making Bluesky data readily available. As of time of writing, we don't have readily available APIs yet. However, ask in the lab to see if this has changed (and this wiki will hopefully be updated to reflect any such changes).

This will hopefully live in the [lab data integrations project](https://github.com/METResearchGroup/lab_data_integrations_interface), so check there first for updates and ask whoever has access to this, for more info (e.g., likely Mark).

## Approach 2: Use the supported APIs

Bluesky offers a few supported APIs. [Here are the developer docs](https://docs.bsky.app/). The Bluesky team only officially supports a Typescript SDK, though an unofficial (but well-performing and just-as-good) [Python SDK](https://atproto.blue/en/latest/) also exists. These are all just wrappers on top of the API, so you can just as well make the requests yourself to the endpoints using any programming language.

Some of these endpoints are unauthenticated and some require a username and password. To make it as easy to use, I'd suggest creating an account and having a username and password available, as these authenticated endpoints tend to be more user-friendly, easier to use, and have higher rate limits.

## Approach 3 (Advanced): Connect to the real-time firehose

Bluesky operates a [firehose](https://atproto.blue/en/latest/atproto_firehose/index.html), which is a real-time data stream. What this means is that whenever anyone posts, likes, follows (or unfollows), deletes a post, or does basically anything on the Bluesky platform, the data stream publishes it and anyone can subscribe to listen to the firehose for updates. Check out [this demo](https://firesky.tv/) for a real-time view of incoming posts through the firehose.

At the time of writing, we are currently working on downloading all the data ourself into a data lake for the lab and building the query layer on top so that anyone can access it, as part of the [lab data integrations project](https://github.com/METResearchGroup/lab_data_integrations_interface). Ideally, check here first to see if you can access any data you need through there. Otherwise, if you're interested in setting up your own custom firehose connection, [these sets of scripts](https://github.com/METResearchGroup/bluesky-research/tree/main/services/sync/stream) are a good starting point.

## Approach 4 (Advanced): Run PDS backfills

Bluesky basically open-sources all their data (which is why it's so useful for academic research). The Bluesky firehose lets us get incoming data as it's being published.

This part requires some deeper understanding of how Bluesky works, as well as comparing it to how tech companies generally work.

First, read the ["How does Bluesky store data?"](WHAT_IS_BLUESKY.md) section, then come back to this part.
