# Phase 1 — Comparison Report

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
| IaaS VM | <!--answer:vm_cold_ms-->284.4<!--/answer--> | <!--answer:vm_warm_median_ms-->209.9<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1768.7<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->637.8<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1653.1<!--/answer--> | <!--answer:function_warm_median_ms-->633.8<!--/answer--> |

<!--answer:latency_plot-->
Phase 1 — deployment measurements (output of measure.py)
------------------------------------------------------------
[OK  ] vm        cold=  284.4ms  warm_median=  209.9ms
            #############
[OK  ] cloudrun  cold= 1768.7ms  warm_median=  637.8ms
            ########################################
[OK  ] function  cold= 1653.1ms  warm_median=  633.8ms
            ########################################
------------------------------------------------------------

For the VM, the cold number is basically the same as the warm one (284 ms vs 210 ms). This
makes sense because the container never stops, so there is no real cold start to measure.
Cloud Run and the Function both show a clear cold cost: around 2.7 times the warm latency.
I think most of that comes from the image pull and the Python interpreter starting up.
Also I noticed that the warm medians are almost identical (638 ms and 634 ms), which was
surprising at first but I explain it in section 4.
<!--/answer-->

---

## 3. Per-platform analysis

<!--answer:scaling_behaviour-->
On the VM nothing happens when the load goes up. I only have one e2-micro and gunicorn
runs with 1 worker and 8 threads, so when more requests arrive they just wait in the
queue. The hard limit is the machine itself, I chose e2-micro to stay in the free tier.
For Cloud Run, Google creates new container instances automatically when traffic grows,
but I capped it at 3 instances with the --max-instances=3 flag. The Function works the
same way, also capped at 3. So once I hit 3 instances on the serverless side, the extra
requests have to wait for one to be free again.
<!--/answer-->

<!--answer:deployment_effort-->
The three platforms are very different in terms of work. The VM took the most time: I had
to create the VPC echo-net, the subnet echo-subnet, two firewall rules (one for port 8080,
one for SSH), a service account with the artifactregistry.reader role, then SSH into the
VM to install Docker with apt, authenticate it to Artifact Registry, pull my image and run
the container. For Cloud Run it was only one command (gcloud run deploy) with the same
image, a region and the min/max instances. Google took care of the HTTPS, the URL and the
scaling. The Function was the same style, one command with --source=. and Google built the
image for me from main.py.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

<!--answer:run_services_list-->
SERVICE: echo
REGION: us-central1
URL: https://echo-696778150088.us-central1.run.app
LAST DEPLOYED BY: service-696778150088@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-11-02T16:29:39.708557Z

SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-696778150088.us-central1.run.app
LAST DEPLOYED BY: baptiste.sardou@eurecom.fr
LAST DEPLOYED AT: 2026-11-02T16:28:36.883879Z
<!--/answer-->

<!--answer:convergence_explanation-->
When I ran gcloud run services list I was surprised to see two services instead of one:
echo-bot (the Cloud Run I deployed myself) and echo (which is supposed to be my Cloud
Function). The Function was deployed by gcf-admin-robot, which is Google internal service
account, but the service lives in the exact same Cloud Run runtime. So basically a Cloud
Function gen2 is just a Cloud Run service with an extra build step on top
(functions-framework wraps main.py as an HTTP server). That is why the warm medians are
almost the same (633.8 ms vs 637.8 ms, only 4 ms of difference) and the cold starts are
close too (1653 ms vs 1769 ms). They run the same type of container on the same scheduler.
The only real difference is how I deploy them: Cloud Run uses a pre-built image I pushed
to Artifact Registry, the Function takes main.py as source and Google builds the image
for me with Cloud Build.
<!--/answer-->
