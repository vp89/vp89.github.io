---
layout: post
title:  "Tips for staging files in S3"
date:   2019-01-05 00:00:00 -0000
categories: programming
---

If you need to process streams of data in batches, one way to do that is to read from the stream and write to an S3 bucket. You then periodically list and process all files in the bucket. S3 is a powerful and useful service but just like anything else, if used incorrectly it can lead to bad performance.

### Minimize file count in your staging bucket

You will need to do an `ls` on your bucket to see what data is ready for processing. The `ls` command grinds to a halt once you get into the range of 100k files and can end up being a significant factor in your overall processing time. You can work around it by using prefixes and multiple concurrent calls but an easier solution is to minimize the number of files in the bucket/prefix at any given time.

When you are reading off the stream, batch your writes to S3 to produce fewer larger files.

After you have processed the file, delete it or move it to another bucket if you have to archive it. If the staging bucket stays small you will have consistent, good performance reading from it. You could then have a compaction job on your archive bucket to make that faster to list/search.

### Name your files based on expected search patterns

In S3 files are sorted lexicographically. One way to make the list operations run faster is to use prefixes in the file names. Depending on your expected access pattern you could name your files using a UUID or a combination of a 3-4 character prefix and a timestamp.

```
transaction/c24836c6-815b-4b52-b217-99f4e5f1b45a-2018-01-01-00-00-00-000.csv
transaction/012-2018-01-01-00-00-00-000.csv
```

The first pattern would allow you super granular list operations up to 32 characters in precision but it's likely that it will be more useful to you to be able to have a 2 or 3 character prefix and timestamp. This would allow you to run the `ls` commands concurrently while still doing fine-grained time-based searches on your bucket.

```
ls s3://bucket/transaction/000-2018-04
ls s3://bucket/transaction/001-2018-04
ls s3://bucket/transaction/002-2018-04
etc ...
```

You can use your expected volume to determine the ideal prefix length. One thing to consider is that S3 allows up to 5500 requests per second to retrieve data so you will need to manage the rate limiting on your end if you want to go over 3 characters.

### Watch out for versioning

If you don't want to move the files to an archive bucket but still need to delete them so they aren't reprocessed, you may think to turn on file versioning on the bucket.

This solution doesn't scale well because deleted files still exist as part of the index. So when you do an `ls`, S3 has to iterate through the deleted files to know that it shouldn't return them to you. You end up getting to a state where you have an "empty" bucket but the `ls` commands take a really long time because it's looping through hundreds of millions of deleted files in the index.
