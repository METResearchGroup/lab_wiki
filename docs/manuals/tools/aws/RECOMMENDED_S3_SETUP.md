# Recommended S3 setup

It's difficult to organize information into S3, and without a consistent setup, it's easy to have files strewn everywhere. What I highly recommend is something like the following:

1. A project-specific bucket: create or use an S3 bucket to store all the information for your specific study.
2. An `experiments/` folder: for any individual experiments that you do, I recommend putting them in the experiments folder. That way, all your different experimental attempts are there. I recommend a setup like `experiments/{short identifier for your experiment}_{date}/` (e.g., `experiments/moral_outrage_classifier_finetuning_2026_09_01/`).
3. A `docs/` folder: for any sort of write-ups or drafts or slide decks, I recommend having a Docs folder in S3 for that, if you want to keep it in there.
