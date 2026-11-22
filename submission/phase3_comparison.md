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
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->31.3<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.014<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->2.1<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->4<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
The local implementation was the fastest (0.014 s), followed by Spark (2.1 s), while cloud MapReduce took the longest (31.3 s). The local and cloud implementations perform the same word-count task, so this is the like-for-like comparison. Cloud execution spends additional time on task scheduling, remote worker invocation, network communication and Cloud Storage operations. Spark performs a more computationally expensive TF-IDF task, involving two `reduceByKey` operations and a `join`, on a single machine. Its execution time suggests that much of the cloud MapReduce time comes from distributed execution overhead rather than computation alone.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
The Spark pipeline has three shuffle boundaries: the first `reduceByKey` aggregates term frequencies, the second computes document frequencies, and the `join` combines TF and DF by term. The cloud workflow has 16 original tasks (12 map and 4 reduce). Four retries correspond to 25% of these tasks, reasonably close to the 30% chaos probability. The difference is expected because failures are injected probabilistically, so the observed retry count does not have to match the configured probability exactly.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
The pipeline uses flatMap to tokenize documents and emit term-document pairs, map to extract terms and prepare intermediate records, reduceByKey to aggregate term and document frequencies, join to combine TF and DF, and another map to calculate TF-IDF values. These are all transformations and are evaluated lazily. Calling build_tfidf_rdd only constructs the RDD lineage without executing the computation. The collect operation is an action: it triggers the execution of the entire pipeline and brings the resulting rows to the driver.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
If an executor holding an output partition of the first `reduceByKey` failed, Spark would recompute the missing partition from its parent RDD using lineage. It could reuse surviving shuffle outputs from the first shuffle stage, avoiding recomputation of unaffected partitions. Recovery therefore starts at the shuffle boundary introduced by the first `reduceByKey`. If some required shuffle data was also lost, Spark would have to recompute the corresponding upstream map tasks.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows (workflow.yaml), which orchestrates the tasks, handles phase transitions and retries.<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->Cloud Run worker service executing the Python code in main.py.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Google Cloud Storage, which stores input splits, intermediate shuffle files and reducer outputs.<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->The partition(word, R) function in the map worker, which assigns each key to a reducer.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is provided by the parallel construct in Cloud Workflows. It waits for all parallel map branches to finish successfully before the workflow proceeds to the reduce phase.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
The worker is idempotent because each task writes its output to a deterministic Cloud Storage object and overwrites the previous content. Therefore, retrying a failed task does not create duplicate results. If run_map appended pairs instead of overwriting the shuffle file, a retry could write the same pairs twice. The reducers would then count duplicate records, producing incorrect word frequencies.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
The MapReduce execution completed successfully, with the final state reported as SUCCEEDED. The chaos parameter was set to 0.3 to simulate intermittent worker failures. Cloud Workflows handles these failures through its retry mechanism. The final word-count result matched the local reference, as confirmed by matches_local being true.
<!--/answer-->
