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

|                                                                         | Value                                           |
| ----------------------------------------------------------------------- | ----------------------------------------------- |
| Map tasks in the cloud job (`mapreduce.num_splits`)                     | <!--answer:num_map_tasks-->12<!--/answer-->     |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`)                | <!--answer:num_reduce_tasks-->4<!--/answer-->   |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`)        | <!--answer:cloud_elapsed_s-->73.4<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`)   | <!--answer:local_elapsed_s-->0.01<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`)    | <!--answer:spark_elapsed_s-->4.8<!--/answer-->  |
| Number of **stages** in the Spark job (read off notebook 2b / 3a)       | <!--answer:num_stages-->4<!--/answer-->         |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->2<!--/answer-->      |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Local (0.01 s) < Spark (4.8 s) < cloud MapReduce (73.4 s). The local run executes map, shuffle and reduce in one process. The cloud run (like-for-like) runs the same word count as HTTP-invoked Cloud Run functions, ~4 orders of magnitude slower due to invocations, Cloud Storage round trips and chaos retries. Spark does more work but beats the cloud job, pipelining in memory and writing only shuffle outputs to local disk: the cloud job's time is overhead, not computation.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
Our Spark run had 4 stages, i.e. 3 shuffles: `reduceByKey` in `term_freq`, `reduceByKey` in `doc_freq`, and the `join` in `build_tfidf_rdd`. The chaos run had 2 retries (two `503`s in Logs Explorer, see screenshot below), so 2 of 18 attempts (≈11%) failed: below the ~30% that `--chaos 0.3` should produce, but plausible for only 16 tasks.
<!--/answer-->

*Logs Explorer, `mr-worker` filtered: the two simulated-crash `503` responses from the chaos run (job `mr-20261112-212008`).*

![Two 503 retries of mr-worker in Logs Explorer](../phase-3-mapreduce-spark/notes/Screenshot%20From%202026-11-12%2022-37-22.png)

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
Transformations (lazy, each returns a new RDD): `parallelize` (creates `docs_rdd`), `flatMap` and `reduceByKey` in `term_freq`, `map` and `reduceByKey` in `doc_freq`, and `map`, `join` and `map(to_row)` in `build_tfidf_rdd`. The only action is `collect` in `build_tfidf_rows`. Running `build_tfidf_rdd` returned instantly: it only recorded the lineage, which `toDebugString` printed. `collect` triggered execution: Spark cut the lineage into 4 stages at the shuffles, ran their tasks and returned 5975 rows to the driver.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
That partition belongs to `ShuffledRDD[3]`, the reduce side of shuffle 1 (`term_freq`'s `reduceByKey`). Spark reruns only that partition's task, refetching its shuffle-1 blocks and re-summing them. Stage 1 (`parallelize` → `flatMap` → the map-side half of `reduceByKey`, which writes the shuffle-1 files) is reused, because those files already exist; it reruns only if they were lost too. Recovery starts at the shuffle-1 boundary.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows: the `mr-wordcount` workflow (`workflow.yaml`)<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->Cloud Run functions (2nd gen): the `mr-worker` function (`main.py`), one invocation per map or reduce task<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage: the `single-odyssey-510214-b4-eurecomgpt` bucket (`<job>/input/`, `<job>/shuffle/`, `<job>/output/`)<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->`partition(word, num_reducers)` in `mapreduce.py` (`crc32(word) % R`), called by `run_map` in `main.py`<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
In the `map_phase` step's `parallel` block: a `parallel` step completes only when all of its `for` branches have returned, so the next step, `reduce_phase`, cannot start until the last map task has finished.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Each task reads its input from Cloud Storage and overwrites fixed, deterministic paths (`shuffle/split-i/part-r.json`, `output/part-r.json`); the worker keeps no other state. Rerunning a task therefore rewrites identical files, so a retry is harmless. If `run_map` appended instead, a task that crashed or timed out after writing some of its files and was then retried would write its pairs twice, and the reducers would double-count those words.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
Logs Explorer, filtered on `mr-worker`, showed two requests from Cloud Workflows answered with `503` (21:20:13 and 21:20:32 UTC): the simulated worker crashes. The workflow retried those tasks with backoff and they then succeeded. The execution finished `SUCCEEDED` in 73.4 s with 2221 distinct words, identical to the local reference (`matches_local: true`).
<!--/answer-->
