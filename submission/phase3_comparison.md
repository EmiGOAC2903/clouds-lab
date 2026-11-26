# Phase 3 — Comparison Writeup

Copy this file to `submission/phase3_comparison.md`, fill in every answer slot, and commit it:

```bash
cp phase-3-mapreduce-spark/comparison_template.md submission/phase3_comparison.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers in section 1 must **match `submission/phase3_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->26.6<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.022<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->5.2<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->6<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
From fastest to slowest: the local single-process word count (0.022 s), Spark TF-IDF (5.2 s) and the cloud MapReduce (26.6 s). The like-for-like pair is local versus cloud MapReduce, since both are the same word count: the cloud job is about 1,200 times slower because every task is an HTTP call to a Cloud Run container that has to start, read and write Cloud Storage objects, and be scheduled by Workflows. 
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
The Spark job had 3 shuffles, which gives 4 stages: the reduceByKey in term_freq, the reduceByKey in doc_freq, and the join in build_tfidf_rdd. In the chaos run, 6 attempts failed with a 503 out of 22 attempts in total (16 tasks plus 6 retries), about 27 %, close to the 30 %.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
flatMap, map, reduceByKey and join are transformations: they only add steps to the lineage and compute nothing. parallelize just creates the first RDD. collect is the only action. Running build_tfidf_rdd returned instantly and only built the lineage, which I printed with toDebugString.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
Spark would not restart the whole job. From the lineage, it would recompute only the lost partition of the first reduceByKey (stage 0), by re-running that one task from its parent, the parallelized documents with the flatMap that splits them into terms. The other partition and the data of later stages that already exist can be reused.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows: the mr-wordcount workflow (workflow.yaml), with parallel/for for the fan-out and try/retry for failures.<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The Cloud Run function mr-worker (main.py), invoked once per map or reduce task, running as the service account mr-runner and calling map_wc, shuffle and reduce_wc.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage, the bucket project-80597ad9-1ebe-4675-93f-eurecomgpt: input splits, one shuffle file per mapper and reducer, and the output partitions.<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->The partition() function in mapreduce.py (zlib.crc32(word) % num_reducers), used by each mapper to decide which reducer file each word goes to.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The parallel step of map_phase: it only finishes when all its branches have finished, so reduce_phase, which comes right after it, starts only after the last map task has returned.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Each task overwrites its own output objects with the same content for the same input, so running it twice after a 503 leaves exactly the same state. If run_map appended its pairs to the shuffle file, a retried mapper would write its pairs twice and the reducers would count those words double, so the cloud result would no longer match the local reference.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
With --chaos 0.3 the job still finished with SUCCEEDED in 26.6 s and the result was identical to the local reference (identical: True). In Logs Explorer, for the mr-worker service, I found 6 requests that returned 503.
<!--/answer-->
