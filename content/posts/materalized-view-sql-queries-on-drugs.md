---
title: "Materalized View: Sql Queries on Steroids"
date: 2022-12-29T01:51:11+04:00
draft: true
---

I have been working with a client with close to 600k images on their home page. The photos are tagged with multiple categories. The index API returns paginated pictures based on various sets of filters on classes.

Recently, they normalized their data, and every image was mapped to 4 different categories through join tables. So to load the home page, the app server used to join 8 tables on runtime and return the results.

Below is the query that filters the images tagged `ABSTRACT` for a page and limit results to 10.

```sql
SELECT DISTINCT ON (images.id) external_uuid
FROM images
         JOIN images_chapters qc ON images.id = qc.image_id
         JOIN chapters c ON c.id = qc.chapter_id
         JOIN images_sections qs ON images.id = qs.image_id
         JOIN sections s ON s.id = qs.section_id
         JOIN images_subjects qs2 ON images.id = qs2.image_id
         JOIN subjects s2 ON s2.id = qs2.subject_id
         JOIN images_topics qt ON images.id = qt.image_id
         JOIN topics t ON t.id = qt.topic_id
WHERE s.name = 'ABSTRACT'
ORDER BY images.id
OFFSET <offset_page> LIMIT 10
```

The count on the actual category tables is meagre <5k rows per table. The join tables have mostly 1(images):1(categories) mapping. Meaning every image has at least been tagged into 4 categories. If our filter predicate results in 100k images, we are essentially joining 5 tables ( images + 4 classes) of 100k each.

Below is the result of `EXPLAIN ANALYZE` that filters the images tagged `ABSTRACT` for a specific page, sorts by `images.id`, and returns the first 10 results.

```sql
Limit  (cost=2256.12..2256.17 rows=10 width=45) (actual time=939.317..939.329 rows=10 loops=1)
  ->  Unique  (cost=2255.87..2257.11 rows=248 width=45) (actual time=939.292..939.324 rows=60 loops=1)
        ->  Sort  (cost=2255.87..2256.49 rows=248 width=45) (actual time=939.291..939.308 rows=64 loops=1)
                Sort Key: images.id
                Sort Method: external merge Disk: 3648kB
              ->  Hash Join  (cost=181.85..2246.01 rows=248 width=45) (actual time=3.048..905.152 rows=64082 loops=1)
                    ->  Nested Loop  (cost=136.55..2200.06 rows=248 width=53) (actual time=2.730..887.310 rows=64082 loops=1)
                          ->  Nested Loop  (cost=136.13..2084.93 rows=236 width=69) (actual time=2.713..704.197 rows=59960 loops=1)
                                ->  Nested Loop  (cost=135.98..2045.35 rows=236 width=77) (actual time=2.701..606.176 rows=59960 loops=1)
                                      ->  Nested Loop  (cost=135.56..1930.55 rows=236 width=61) (actual time=2.683..432.067 rows=59960 loops=1)
                                            ->  Hash Join  (cost=135.14..1762.99 rows=237 width=16) (actual time=2.666..218.527 rows=59960 loops=1)
                                                  ->  Nested Loop  (cost=129.53..1756.48 rows=334 width=24) (actual time=2.609..202.749 rows=59960 loops=1)
                                                        ->  Nested Loop  (cost=129.12..1595.68 rows=329 width=8) (actual time=2.589..25.415 rows=59336 loops=1)
                                                              ->  Index Scan using index_sections_on_name on sections s  (cost=0.15..8.17 rows=1 width=8) (actual time=0.014..0.015 rows=1 loops=1)
                                                                      Index Cond: ((name)::text = 'ABSTRACT'::text)
                                                              ->  Bitmap Heap Scan on images_sections qs  (cost=128.97..1519.71 rows=6780 width=16) (actual time=2.571..16.893 rows=59336 loops=1)
                                                                      Recheck Cond: (section_id = s.id)
                                                                      Heap Blocks: exact=1300
                                                                    ->  Bitmap Index Scan on index_images_sections_on_section_id  (cost=0.00..127.27 rows=6780 width=0) (actual time=2.442..2.442 rows=59568 loops=1)
                                                                            Index Cond: (section_id = s.id)
                                                        ->  Index Scan using index_images_chapters_on_image_id on images_chapters qc  (cost=0.42..0.48 rows=1 width=16) (actual time=0.002..0.002 rows=1 loops=59336)
                                                                Index Cond: (image_id = qs.image_id)
                                                  ->  Hash  (cost=4.16..4.16 rows=116 width=8) (actual time=0.050..0.050 rows=171 loops=1)
                                                        ->  Seq Scan on chapters c  (cost=0.00..4.16 rows=116 width=8) (actual time=0.005..0.025 rows=171 loops=1)
                                            ->  Index Scan using images_pkey on images images  (cost=0.42..0.71 rows=1 width=45) (actual time=0.003..0.003 rows=1 loops=59960)
                                                    Index Cond: (id = qc.image_id)
                                      ->  Index Scan using index_images_subjects_on_image_id on images_subjects qs2  (cost=0.42..0.48 rows=1 width=16) (actual time=0.002..0.002 rows=1 loops=59960)
                                              Index Cond: (image_id = images.id)
                                ->  Index Only Scan using subjects_pkey on subjects s2  (cost=0.15..0.17 rows=1 width=8) (actual time=0.001..0.001 rows=1 loops=59960)
                                        Index Cond: (id = qs2.subject_id)
                                        Heap Fetches: 59960
                          ->  Index Scan using index_images_topics_on_image_id on images_topics qt  (cost=0.42..0.48 rows=1 width=16) (actual time=0.002..0.002 rows=1 loops=59960)
                                  Index Cond: (image_id = images.id)
                    ->  Hash  (cost=32.91..32.91 rows=991 width=8) (actual time=0.311..0.311 rows=1035 loops=1)
                          ->  Seq Scan on topics t  (cost=0.00..32.91 rows=991 width=8) (actual time=0.007..0.155 rows=1035 loops=1)
Planning time: 30.718 ms
Execution time: 941.265 ms
```

```sql
SELECT DISTINCT ON (id) external_uuid
FROM materialized_view_images
WHERE section_name = 'ABSTRACT'
ORDER BY id
OFFSET <offset_page> LIMIT 10
```
