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
The ranking is very clear, here are the data that I found : local 0.011s, Spark 4.8s, cloud MapReduce 19.9s. The local run is by far the fastest one beacause the rest stays in one process. Spark do more work, but end up in 4.8 seconds because it runs on the sae machine Colab without network between tasks. The cloud MapReduce is the slowest because each task is an invocation of the cloud run function with cold start, atuh and eah shuffle go through cloud storage. Spark and local are not comparable but local and cloud MapReduce yes. It's the same. Indeed, we have the same code, and documents but it's x1800 slower just because of the overcost.
<!--/answer-->

<!--answer:counts_interpretation-->
Spark had 4 stages so 3 frontiers of shuffle. The shuffles come from two queries reduceByKey in term_frq and doc_freq pus the join at the end. For the chaos run with 16 task attempts and chaos = 0,3, about 5 attemps failed and have been retried, which corresponds to a "coef" of 30%.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

<!--answer:transformations_vs_actions-->
In spark_tfidf.py, flatMap, map, reduceByKey and join are all transformations, they are lazy. In the notebook, build_tfidf_rdd only build the graph of lineage, but nothing executes. Then, save_parquet stocks correcly without any computation. The real action is collect in the cell 2c where it triggers Spark to really execute all the DAG and output the rows to the driver.
<!--/answer-->

<!--answer:lineage_recovery-->
If the worker who has a partition of the first reduceByKey died, Spark would only recompute this lost partition. It would re-read the input from ParallelCollectionRDD, re-execute flatmap and the reduce partial for the parition only. The recuperation starts at the frontier of the shuffle of stage 0.
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
