# Phase 3 — Comparison Writeup

---

## 1. Facts from your runs

| | Value |
|---|---|
| Map tasks in the cloud job | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds | <!--answer:cloud_elapsed_s-->19.9<!--/answer--> |
| Local single-process wall time, seconds | <!--answer:local_elapsed_s-->0.011<!--/answer--> |
| Spark wall time for the collect, seconds | <!--answer:spark_elapsed_s-->4.8<!--/answer--> |
| Number of stages in the Spark job | <!--answer:num_stages-->4<!--/answer--> |
| Task retries you counted in your --chaos run | <!--answer:chaos_retries-->5<!--/answer--> |

<!--answer:timing_explanation-->
The ranking is very clear, here is the data I find : local 0.011s, Spark 4.8s, cloud MapReduce
19.9s. The local run is the fastest by far because everything stays in one process. Spark does
more work , but it still finishes in 4.8s because it runs on the same Colab machine with no
network between tasks. The cloud MapReduce is the slowest because each task is a Cloud Run
function invocation with cold start, auth, and every shuffle write/read goes through Cloud
Storage. Spark and local are not like-for-like, but local vs cloud MapReduce are the same.
Indeed we have the same code, the same 60 documents, but 1800x slower just because of
distributed overhead.
<!--/answer-->

<!--answer:counts_interpretation-->
Spark had 4 stages so 3 shuffle boundaries. The shuffles came from the two reduceByKey calls
in term_freq and doc_freq, plus the join at the end. For the chaos run with 16 task attempts
and chaos=0.3, about 5 attempts failed and were retried, which matches the 30% rate.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

<!--answer:transformations_vs_actions-->
In spark_tfidf.py flatMap, map, reduceByKey and join are all transformations, they are lazy.
In the notebook build_tfidf_rdd only builds the lineage graph, nothing runs. Then save_parquet
also just stores, no compute. The real action is collect in cell 2c, that is what triggers
Spark to actually run the whole DAG, read the data, do the 4 stages and return the 5975 rows
to the driver.
<!--/answer-->

<!--answer:lineage_recovery-->
If the worker with one partition of the first reduceByKey died, Spark would only recompute
that one lost partition.. It would re-read the input from ParallelCollectionRDD, re-run
flatMap and the partial reduce for that partition only. Recovery starts at the shuffle
boundary of stage 0
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

- **JobTracker:** <!--answer:role_jobtracker-->Cloud Workflows (workflow.yaml mr-wordcount)<!--/answer-->
- **Workers:** <!--answer:role_workers-->Cloud Run function mr-worker (main.py)<!--/answer-->
- **HDFS:** <!--answer:role_hdfs-->Cloud Storage bucket eurecomgpt2-513516-eurecomgpt<!--/answer-->
- **Partitioner:** <!--answer:role_partitioner-->partition() in mapreduce.py (zlib.crc32(word) % num_reducers)<!--/answer-->

<!--answer:barrier_location-->
The barrier is the semantics of the parallel block itself: a parallel step only completes
when every branch inside it finishes. So reduce_phase cannot start before map_phase is done.
<!--/answer-->

<!--answer:idempotence-->
The worker overwrites its shuffle file every time it runs. So if the same map task runs
twice the second run produces the same output and replaces the first one. There is no
duplication. If run_map appended instead of overwriting, a retried task would write its
pairs on top of the partial pairs from the failed attempt, and the reducer would count some
words twice. The output would not match the local reference anymore.
<!--/answer-->

<!--answer:chaos_observations-->
In the execution log I could see some task branches being retried after returning 503. Logs
Explorer showed simulated worker crash entries for the mr-worker function. Despite roughly 5
retried attempts the job still finished with state SUCCEEDED in 19.9s and matches_local was
still true, so the final counts were identical to the local reference.
<!--/answer-->
