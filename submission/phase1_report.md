# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
49.99
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->239.6<!--/answer--> | <!--answer:vm_warm_median_ms-->209.4<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->3218.1<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->637.4<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->680.3<!--/answer--> | <!--answer:function_warm_median_ms-->638.5<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```text
Warm latency distribution (20 requests per platform)
Each # represents one request.

VM:
200-299 ms | #################### (20)

Cloud Run:
100-199 ms | ### (3)
600-699 ms | ################# (17)

Cloud Function:
100-199 ms | ##### (5)
600-699 ms | ############# (13)
700-799 ms | ## (2)
```
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
The VM does not scale automatically. All requests use the same e2-micro machine, so its CPU and memory limit performance.

Cloud Run automatically creates more instances when traffic increases. However, I configured a maximum of 3 instances. It can also scale down to zero when idle.

Cloud Functions Gen 2 also scales automatically using Cloud Run infrastructure. I configured a minimum of zero instances, while its maximum capacity depends on service settings and resource quotas.

Therefore, Cloud Run and Cloud Functions handle increasing traffic more easily than a single VM.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
Deploying on the VM required the most manual work. I created a VPC, subnet, and firewall rules, created an e2-micro VM, connected with SSH, installed Docker, pulled the image, and started the container. I also need to maintain the operating system.

Cloud Run was simpler. I only needed to deploy the Docker image and configure instance limits and public access. Google manages the infrastructure.

Cloud Functions Gen 2 was also simple. I deployed the Python function and configured HTTP access and build permissions. I did not need to install Docker or manage a virtual machine.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```text
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-845392248886.us-central1.run.app
LAST DEPLOYED BY: service-845392248886@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-23T18:03:56.756596Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-845392248886.us-central1.run.app
LAST DEPLOYED BY: Xin.Huang@eurecom.fr
LAST DEPLOYED AT: 2026-10-23T17:20:08.582367Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The service list shows two Cloud Run services: echo-bot, deployed directly from a Docker image, and echo, created by the second-generation Cloud Function deployment.

This shows that Cloud Functions Gen 2 uses Cloud Run infrastructure. Both services therefore share similar container-based execution and scaling mechanisms.

In my measurements, their warm median latencies were almost identical: 637.4 ms for Cloud Run and 638.5 ms for Cloud Function.

However, their measured cold latencies differed: 3218.1 ms and 680.3 ms respectively. This difference may depend on instance state and network conditions.

The main difference is deployment: Cloud Run uses a container image, while Cloud Functions deploys Python function source code and manages the underlying service automatically.
<!--/answer-->
