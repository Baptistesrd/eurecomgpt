# Phase 1 — Comparison Report

> **Note to instructor:** Phase 1 was deployed using my personal Google Cloud account
> (babasardou@gmail.com, project eurecomgpt) instead of my Eurecom account. My Eurecom
> account (baptiste.sardou@eurecom.fr) was locked by Google since Thursday Oct 23 — Google
> began enforcing 2-Step Verification on Oct 20 and systematically refused to verify me
> ("We couldn't verify it's you") on every attempt, both from PC and mobile, even after
> adding a recovery phone. This made it impossible to enable the Compute/Artifact Registry
> APIs or link a billing account to the Eurecom project. I emailed you on Monday Oct 26 to
> request approval; multiple other students reported the same issue in the class group chat.
> All deployments, measurements, and teardown were performed identically to spec — only the
> hosting GCP project changed.

---

## 1. Setup facts

**Your image size**, in MB:

<!--answer:image_size_mb-->
50
<!--/answer-->

**The custom VPC and subnet you created**:

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->254.4<!--/answer--> | <!--answer:vm_warm_median_ms-->212.4<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->129.9<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->146.5<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1164.9<!--/answer--> | <!--answer:function_warm_median_ms-->131.6<!--/answer--> |

<!--answer:latency_plot-->
Phase 1 — deployment measurements (output of measure.py)
------------------------------------------------------------
[OK  ] vm        cold=  254.4ms  warm_median=  212.4ms
            ########################################
[OK  ] cloudrun  cold=  129.9ms  warm_median=  146.5ms
            ############################
[OK  ] function  cold= 1164.9ms  warm_median=  131.6ms
            #########################
------------------------------------------------------------

The VM shows no meaningful cold start (254 ms vs 212 ms warm): its container never stops, so
every request is effectively warm. Cloud Run's cold (130 ms) is actually lower than its warm
median (147 ms), because Google kept an instance warm from recent testing — the "cold" sample
here is not a true cold start. The Function is the only one showing a real cold-start cost:
1165 ms vs 132 ms warm, an 8.8x penalty, dominated by image pull and Python interpreter boot.
<!--/answer-->

---

## 3. Per-platform analysis

<!--answer:scaling_behaviour-->
The IaaS VM creates nothing: a single e2-micro instance handles everything, extra requests
queue behind gunicorn's 1 worker x 8 threads, and the ceiling is the machine itself
(--machine-type=e2-micro). Cloud Run auto-provisions new container instances on demand,
one per concurrent request burst, capped at 3 by the --max-instances=3 flag I set at
deploy. The Cloud Function behaves identically — it spins up fresh instances to absorb load,
also capped at 3 by my --max-instances=3. Beyond that cap, both serverless targets queue
requests until an instance frees up.
<!--/answer-->

<!--answer:deployment_effort-->
The three platforms demanded wildly different amounts of work. The VM required the most: a
custom VPC (echo-net) and subnet (echo-subnet), two firewall rules (allow-echo-8080 for the
app, allow-echo-ssh for access), a dedicated service account with roles/artifactregistry.reader,
then SSH into the instance to apt-install Docker, authenticate it against Artifact Registry,
pull the image, and run it detached with --restart=always. Cloud Run was a single
gcloud run deploy command: pass the same image, pick a region, set min/max instances. Google
handled networking, HTTPS, scaling and a public URL. The Function was similarly one-shot:
gcloud functions deploy --source=. and Google built, hosted and routed everything from main.py.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

<!--answer:run_services_list-->
SERVICE: echo
REGION: us-central1
URL: https://echo-748214485908.us-central1.run.app
LAST DEPLOYED BY: service-748214485908@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-27T10:47:55.424943Z

SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-748214485908.us-central1.run.app
LAST DEPLOYED BY: babasardou@gmail.com
LAST DEPLOYED AT: 2026-10-27T10:46:11.024671Z
<!--/answer-->

<!--answer:convergence_explanation-->
gcloud run services list shows two Cloud Run services, not one: echo-bot (the one I deployed
with gcloud run deploy) and echo (my supposedly separate Cloud Function). The Function was
deployed by gcf-admin-robot — Google's internal service account — but it lives in the exact
same Cloud Run runtime. Cloud Functions gen2 is, under the hood, Cloud Run with a thin build
layer (functions-framework wrapping main.py into an HTTP server). That is why the warm
medians converge (Function 131.6 ms vs Cloud Run 146.5 ms): both execute the same
gunicorn-equivalent container on the same scheduler. What genuinely differs is only the
deployment path: Cloud Run takes a pre-built image I pushed to Artifact Registry; Cloud
Functions takes main.py source and builds the image for me via Cloud Build.
<!--/answer-->