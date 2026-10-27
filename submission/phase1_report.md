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
| IaaS VM | <!--answer:vm_cold_ms-->252.9<!--/answer--> | <!--answer:vm_warm_median_ms-->211.7<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2421.6<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->651.2<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1705.8<!--/answer--> | <!--answer:function_warm_median_ms-->641.8<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
```
vm (warm, n=20, median=211.7 ms)
    200-249   ms | #################### 20

cloudrun (warm, n=20, median=651.2 ms)
    100-149   ms | ####### 7
    600-649   ms | ## 2
    650-699   ms | ########### 11

function (warm, n=20, median=641.8 ms)
    100-149   ms | ######## 8
    600-649   ms | ####### 7
    650-699   ms | ##### 5
```
<!--/answer-->

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
IaaS VM: nothing is created automatically. There is one e2-micro VM running one gunicorn worker with 8 threads, so at ten or a hundred requests at once they queue up, get slower and can fail. The limit is the machine size I chose, to scale I would have to create more VMs and a load balancer myself.
Cloud Run: new container instances of my image are started automatically when load grows, and it scales back to zero when idle. Each instance handles up to 80 concurrent requests by default, and the hard limit is the flag I passed, --max-instances=3.
Cloud Function: new instances are also created automatically, but each handles only one request at a time, so a burst of 100 needs many more instances. Its limit is the function's max-instances setting.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
IaaS VM: the most work. I built the image, pushed it to Artifact Registry, created a custom VPC (echo-net), a subnet (echo-subnet), two firewall rules applied through the echo-http network tag, then created the e2-micro VM and ran the container on it. I am responsible for the OS, patches, restarts and the external IP, and it only serves plain HTTP.
Cloud Run: one gcloud run deploy command reusing the same image. Google handles HTTPS, scaling and the servers.
Cloud Function: no Dockerfile needed; I deployed the source (main.py) and Google built the container itself. Easy to write, but the deploy took about 7 minutes compared with under 1 minute for Cloud Run.

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
URL: https://echo-542909156415.us-central1.run.app
LAST DEPLOYED BY: service-542909156415@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-27T11:20:15.391682Z


✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-542909156415.us-central1.run.app
LAST DEPLOYED BY: carlos.lugo-torres@eurecom.fr
LAST DEPLOYED AT: 2026-10-27T11:11:28.264738Z
```
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output lists two Cloud Run services: echo-bot (my container deployment) and echo (my 2nd gen Cloud Function, automatically deployed by gcf-admin-robot), proving 2nd gen functions run on Cloud Run. With min-instances=0, both suffer cold start delays (1705.8 ms and 2421.6 ms) compared to the always-on VM (252.9 ms). The key differences are build and runtime defaults: the function builds from source via gcf-artifacts, uses smaller default resources (0.1666 vCPU, 256 MB), processes only one request per instance (versus 80 on Cloud Run), and takes longer to deploy. Finally, the VM's lower warm latency is likely due to direct HTTP connections, whereas Cloud Run and Cloud Functions route through Google's HTTPS front end. Note that cold start figures represent single-point measurements.
<!--/answer-->
