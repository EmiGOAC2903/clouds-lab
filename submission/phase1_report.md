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
50
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
| IaaS VM | <!--answer:vm_cold_ms-->253.4<!--/answer--> | <!--answer:vm_warm_median_ms-->211.6<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2232.6<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->642.0<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1612.1<!--/answer--> | <!--answer:function_warm_median_ms-->638.2<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```
[OK  ] vm        cold=  253.4ms  warm_median=  211.6ms
            #############
[OK  ] cloudrun  cold= 2232.6ms  warm_median=  642.0ms
            ########################################
[OK  ] function  cold= 1612.1ms  warm_median=  638.2ms
            ########################################
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
The VM creates nothing automatically, it's a single e2-micro machine, so ten or a hundred simultaneous requests all land on the same box and just queue or slow down. Its hard limit is the size of the machine I picked. Cloud Run starts newcontainer instances automatically when load grows, and the hard limit is the flag I passed myself. 
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
The VM took by far the most work, I created a custom VPC, a subnet and two firewall rules by using a network tag, I also created the VM with a service account allowed to pull from Artifact Registry, SSH'd in, installed Docker, authenticated it and ran the container. I also own the OS, the Docker restarts and the external IP cost. Cloud Run needed a Dockerfile and an image pushed to Artifact Registry, with this my local push failes, so I used the Cloud Build, then a single gcloud run deploy command, with no network or machine to manage, The function was the easiest, because no Dockerfile and no image of my own, just main.py, app.py and requirements plus one gcloud functions deploy command, and Google built the container for me.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
```
SERVICE: echo
REGION: us-central1
URL: https://echo-624735392920.us-central1.run.app
LAST DEPLOYED BY: service-624735392920@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-26T13:13:23.837758Z

SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-624735392920.us-central1.run.app
LAST DEPLOYED BY: Emilio.Gonzalez-Acosta@eurecom.fr
LAST DEPLOYED AT: 2026-10-26T13:07:00.553895Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output shows two Cloud Run Services, even though I only deployed one with run deploy. The second one, echo is my Cloud Function, deployed by the Cloud Functions service account. So a gen-2 function runs on Cloud Run underneath, which is why its numbers look so much like Cloud Run's, it's the same platform starting a container that scales to zero, so most of the cold start is the same container startup. What really differs is the deployment path and the defaults. For the Function, Google build the image from my source and wraps my code with the functions framewrok. For Cloud Run, I wrote the Dockerfile, built the image and control the server and the concurrency.
<!--/answer-->
