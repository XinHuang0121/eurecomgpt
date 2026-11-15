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
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->17.1<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.016<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->2.8<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->8<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
The final report ranks local word count (0.016 s) fastest, Spark TF-IDF (2.8 s) next, and cloud MapReduce (17.1 s) slowest. Only the two word counts are like-for-like. Local execution avoids network and scheduling overhead. Cloud execution includes function requests, Cloud Storage I/O, workflow orchestration, retries, and client polling. Spark performs more work on one machine, including two aggregations and a join, yet finishes sooner than cloud word count. This illustrates overhead dominating this tiny corpus, rather than proving a general Spark speedup. These timings are the final report values; the first observed Spark collect took 3.8 s.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
The first Spark job showed four stages and 12 tasks, with three distinct shuffles: term-frequency reduceByKey, document-frequency reduceByKey, and join. ShuffledRDD[3] appears twice in the lineage but is shared. Eight of 16 logical tasks required one retry (50%); eight of 24 total attempts failed (33.3%). The latter is close to the configured 30% failure probability, with normal small-sample variation.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
flatMap, map, reduceByKey, and join are RDD transformations. parallelize creates the source RDD. build_tfidf_rdd assembled the dependency graph without executing the TF-IDF tasks, and toDebugString displayed that lineage. collect is the action that scheduled stages and tasks and returned all 5,975 rows to the driver; build_tfidf_rows also invokes collect. save_parquet subsequently writes the collected Python rows using PyArrow, rather than performing another Spark action.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
Recovery follows the first reduceByKey boundary, ShuffledRDD[3], between stages 0 and 1 in the observed job. Spark can rebuild a lost TF partition by reading surviving stage-0 shuffle blocks and redoing its aggregation. If those blocks were also lost, their producing tasks must be rerun from the source lineage. Unaffected shuffle outputs can be reused; dependent work is recomputed as needed. This is a recovery explanation, not an observed Spark worker-failure experiment.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows, using workflow.yaml deployed as mr-wordcount, orchestrated parallel tasks, the phase barrier, and retries.<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The mr-worker Cloud Run function (second generation), with main.py dispatching map and reduce tasks to the primitives in mapreduce.py.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage bucket eurecomgpt-lab-eurecomgpt held the job input/, shuffle/, and output/ objects; it served the shared-storage role.<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->partition() in mapreduce.py used zlib.crc32(word.encode("utf-8")) % num_reducers to assign each word consistently to a reducer.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The map_phase parallel-for step completes only after all its branches finish successfully, including retries. Sequential workflow execution then starts reduce_phase. The observed map phase ended and reduce phase began at 16:51:02 CET.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Each task uses deterministic object paths for its split or partition and overwrites its outputs. Repeating a task with the same input therefore produces the same stored result instead of adding another contribution. If run_map appended pairs, a retry after a partial write or lost response could duplicate emitted words and inflate reducer counts. The injected chaos failure occurs before work, so this run did not test partial-write failures.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
With chaos 0.3, the Workflows Steps view showed seven map retries and one reduce retry; a failed call reported HTTP 503. Cloud Logging request logs, retrieved through gcloud rather than Logs Explorer, contained eight 503 responses and 16 successful 200 responses. The job finished SUCCEEDED in 17.1 s, and the driver reported identical: True with 2,221 distinct words. Failed try wrappers and their child calls were counted once, not twice.
<!--/answer-->
