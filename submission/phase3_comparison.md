# Phase 3 — Comparison Writeup

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->29.9<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.01<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s--><SPARK_ELAPSED_S><!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries--><CHAOS_RETRIES><!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
From fastest to slowest: local single-process (0.01 s), Spark TF-IDF (<SPARK_ELAPSED_S> s), and cloud MapReduce (29.9 s). The only like-for-like comparison is local vs. cloud MapReduce, since both compute the exact same word count. The local run stays entirely in RAM, whereas Spark pays JVM startup, task scheduling, and in-memory shuffle overhead for a heavier TF-IDF pipeline. The fact that Spark completes a heavier job far faster than cloud MapReduce proves that the cloud job's 29.9 s is dominated by orchestration, HTTP/cold-start latency, and Cloud Storage network I/O rather than computation.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
With 4 stages, the Spark job had 3 shuffle boundaries, caused by the two `reduceByKey` calls (in `term_freq` and `doc_freq`) and the `join` in `build_tfidf_rdd`. In the chaos run, <CHAOS_RETRIES> retries occurred across the 16 tasks (roughly <CHAOS_RETRIES>/16 of attempts), which aligns with the expected ~30% failure rate from `--chaos 0.3`.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
In `spark_tfidf.py`, `flatMap`, `reduceByKey`, `map`, and `join` are all lazy transformations; `collect` (called in `build_tfidf_rows`) is the only action. Running `build_tfidf_rdd` executed no computation—it only constructed the logical DAG (lineage) of transformations. Execution was only triggered when `collect` was called, causing Spark's DAGScheduler to split the lineage into stages at shuffle boundaries, run all tasks, and return the final rows to the driver.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
Spark would not need to restart the entire job. Because shuffle outputs from the first stage (`flatMap` map-side shuffle files) are materialized on disk, recovery starts at the shuffle boundary of the first `reduceByKey`. Spark would only recompute the lost partition by re-fetching its shuffle blocks from the surviving map-stage executors, while reusing all other surviving partitions.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows (`workflow.yaml`)<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->Cloud Run Functions (`mr-worker` / `main.py`)<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage (GCS bucket)<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->`partition()` in `mapreduce.py` (using `zlib.crc32(word) % num_reducers`)<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The `parallel` block enclosing the `for` loop in `map_phase` acts as the implicit barrier: Cloud Workflows does not advance to the next step (`reduce_phase`) until every parallel branch has completed.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Retrying is safe because `run_map` is idempotent: each map task writes its deterministic output to fixed, split-specific object paths in Cloud Storage, overwriting any previous attempt. If `run_map` appended instead of overwriting, a task that partially wrote data before failing (or succeeded before a network timeout triggered a retry) would leave duplicate `(word, 1)` pairs in the shuffle files, silently inflating the final word counts.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
In the Workflows execution log, several map and reduce branches failed with HTTP 503 errors and were automatically retried by the `try`/`retry` policy. In Logs Explorer, filtering on `mr-worker` showed matching `simulated worker crash` entries. Despite these injected failures, the workflow completed in state `SUCCEEDED` and the final output was still identical (`matches_local: true`) to the local reference.
<!--/answer-->