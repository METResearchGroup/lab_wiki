# What is S3?

S3 is a large file storage system in AWS that essentially provides close to infinite file storage for any sort of file you may possibly want. This should be a default go-to source whenever you have data that you want to store for long periods of time.

The way that S3 works is that basically, you have:

1. An S3 bucket. This is like the highest level folder (eg. the "Documents" or "Downloads" folders in your computer)
2. An S3 prefix. This is the subfolder + actual filename for your file (eg "research/file.json")

When you upload/download anything to/from S3, you'll need to specify both of these values.

You also need to choose an S3 bucket that already exists, or create one on the S3 website.

Log into the lab AWS account, using the instructions from NU IT (follow the general access instructions): https://www.it.northwestern.edu/support/login/aws.html

Once logged in, look up "S3" in the console. This should give you a list of buckets that exist already.
