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
SIZE: 50064929 Mb
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
| IaaS VM | <!--answer:vm_cold_ms-->249.7<!--/answer--> | <!--answer:vm_warm_median_ms-->211.9<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2302.3<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->633.1<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1365.1<!--/answer--> | <!--answer:function_warm_median_ms-->632.7<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
Phase 1 — deployment measurements
------------------------------------------------------------
[OK  ] vm        cold=  249.7ms  warm_median=  211.9ms
            #############
[OK  ] cloudrun  cold= 2302.3ms  warm_median=  633.1ms
            ########################################
[OK  ] function  cold= 1365.1ms  warm_median=  632.7ms
            ########################################
------------------------------------------------------------
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
The Iaas VM doesn’t create anything automatically because it has to fit in the ‘e2-micro’ machine size. Cloud Run automatically creates new containers to handle requests arriving at the same time. The number of containers is chosen by ‘—max-instances=3’. Cloud Functions also creates containers and is limited by the default google cloud settings.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
For the VM, I had to build a custom VPC, configure a subnet, open a firewall, install Docker and run the container. To use Cloud Run, it was necessary to build and push a Docker image to Artifact Registry, but the service simplified the processes relating to networking and infrastructure provision into a single command for managed deployment. With Cloud Functions the effort required was minimal since we deployed directly from the source code, Google taking full responsibility for containerisation and for managing the runtime environment.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-77470608751.us-central1.run.app
LAST DEPLOYED BY: service-77470608751@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-11-08T17:26:24.392625Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-77470608751.us-central1.run.app
LAST DEPLOYED BY: gabrieldefombelle59@gmail.com
LAST DEPLOYED AT: 2026-11-08T17:21:12.081317Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output shows that the Cloud Function (echo) is actually a Cloud Run service. As Gen 2 Cloud Functions run on top of the Cloud Run platform they share the same container lifecycle and same underlying execution environment which is why their cold start latencies are so similar. 

The real difference is in the developer experience and the level of deployment abstraction. For Cloud Run (echo-bot) which was deployed directly by my user account I had to write out a Dockerfile, build the container image and push it to Artifact Registry. The Cloud Function on the other hand was deployed by a Google service account (gcf-admin-robot). I just gave it the raw Python source code and Google did all the containerisation and deployment for me.
<!--/answer-->
