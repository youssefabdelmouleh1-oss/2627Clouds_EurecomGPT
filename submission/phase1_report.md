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
49
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
| IaaS VM | <!--answer:vm_cold_ms-->249.0<!--/answer--> | <!--answer:vm_warm_median_ms-->209.8<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2306.0<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->131.4<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1378.1<!--/answer--> | <!--answer:function_warm_median_ms-->392.0<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```
IaaS VM — warm latency, 20 requests (ms)
    208.7 –   209.5 | ######## 8
    209.5 –   210.3 | #### 4
    210.3 –   211.1 | ### 3
    211.1 –   211.9 | # 1
    211.9 –   212.7 | # 1
    212.7 –   213.5 | ### 3

Cloud Run — warm latency, 20 requests (ms)
    119.6 –   218.2 | ############# 13
    218.2 –   316.8 |  0
    316.8 –   415.4 |  0
    415.4 –   514.1 |  0
    514.1 –   612.7 |  0
    612.7 –   711.3 | ####### 7

Cloud Function — warm latency, 20 requests (ms)
    122.1 –   216.8 | ########## 10
    216.8 –   311.5 |  0
    311.5 –   406.1 |  0
    406.1 –   500.8 |  0
    500.8 –   595.5 |  0
    595.5 –   690.2 | ########## 10
```

The VM has no real cold start: its container runs all the time, so its "cold" number is just the first request of the run.
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
VM: nothing gets created. One shared-core e2-micro (1 GB RAM) runs a single container with gunicorn (1 worker, 8 threads), so ten or a hundred requests at once simply queue on that one box. The limit is the machine I picked (--machine-type=e2-micro); scaling out would mean adding VMs and a load balancer myself. Cloud Run: new instances of my container are started automatically, each taking up to 80 concurrent requests, and they scale back to zero when idle. The hard cap is the --max-instances=3 I passed. Function: new instances too, but each handles only one request at a time (concurrency 1), so a burst of 100 asks for up to 100 small instances. I passed no flag, so the cap is the platform default of 100.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
VM: by far the most work. I built a custom VPC and subnet and two firewall rules (8080 for the app, 22 for SSH) that only apply to instances tagged echo-http, so the VM had to carry that tag or the VPC's implied deny-all-ingress would block it. Then I created the VM, gave its service account read access to Artifact Registry, SSHed in, installed and authenticated Docker and ran the container with a restart policy. After that, OS updates, Docker, restarts and the billed external IP are all mine to manage. Cloud Run: I wrote the multi-stage Dockerfile and pushed the image, then one deploy command (port, min/max instances, public access); Google handles HTTPS, load balancing, scaling and the OS. Function: no Dockerfile at all, just main.py and requirements.txt deployed from source. The only snag was granting the build service account the Cloud Build builder role.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-1069887135411.us-central1.run.app
LAST DEPLOYED BY: service-1069887135411@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-26T19:53:57.971868Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-1069887135411.us-central1.run.app
LAST DEPLOYED BY: youssef.abdelmouleh@eurecom.fr
LAST DEPLOYED AT: 2026-10-26T19:46:55.793148Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The list shows two Cloud Run services: echo-bot, which I deployed myself, and echo, deployed by Google's gcf-admin-robot account. That second one is my function. Cloud Run functions (gen 2) run on Cloud Run: Cloud Build packaged main.py with Google's buildpacks and the Functions Framework into a container image (in the gcf-artifacts repository) and deployed it as a Cloud Run service. So both cold starts are the same event, Cloud Run starting an instance from zero, pulling an image and booting Python: 1378.1 ms for the function and 2306.0 ms for Cloud Run, against 249.0 ms for the VM, whose container never stops. With a single cold sample each, I wouldn't read much into the gap between the two. What genuinely differs is packaging and defaults. On Cloud Run I own the Dockerfile and the image, while Google builds the function's container. And the function got 0.17 vCPU and 256 MB with one request per instance, versus 1 vCPU, 512 MiB and concurrency 80 for echo-bot. That smaller CPU share likely explains its slower warm median (392.0 ms vs 131.4 ms).
<!--/answer-->
