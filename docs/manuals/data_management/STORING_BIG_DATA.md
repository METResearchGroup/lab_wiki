# How do you manage big data?

What happens if you're working with big data?

Here, we'll just define big data as "data that's too large to comfortably fit on your computer".

The data files themselves are just too large to fit on your computer. If you're working with at least 10GB of data for a single project, then it might start to get uncomfortably large to fit on your computer.

## Defining the problem

...

## Defining the solution

### Solution 1: Storing as little data as possible on your computer

If you are obtaining the files from somewhere else, just try to load and store only as many files as you need at a time.

There are some use cases where you could run your scripts on one data file at a time. Let's say, for example, that you have a bunch of .csv files for 2025, one for each day. Let's say that storing all the .csv files is too large on your computer.

One strategy is:

1. Load one .csv file at a time (or a few at a time) into your computer.
2. Run your code on those .csv files and store the outputs.
3. Delete these .csv files and load the next batch.
4. Do this until you've gone through all the files.

Some cases where this works well include:

1. Running summary statistics: you can calculate statistics at a granular level and then rederive the larger quantities. For example, if you need an annualized average, you can take the daily averages and the total observations per day and then use that to compute an annualized average.
2. Deriving intermediate outputs (e.g., LLM labels): if you're running code that generates intermediate outputs (e.g., running an LLM classifier on text), you can run this on a small subset of files at a time.

Some cases where this works less well include:

1. Work that requires using the entire dataset: for example, if you're doing a clustering model, you do need all of the embeddings in memory, and loading millions of embeddings is impossible on an average laptop.
2. Work that ...

### Solution 2: Storing data in Quest

...

### Solution 3: Storing data in AWS

...
