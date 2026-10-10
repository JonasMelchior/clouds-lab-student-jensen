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
49.98
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo_net
<!--/answer-->

<!--answer:subnet_name-->
ehco-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->TODO<!--/answer--> | <!--answer:vm_warm_median_ms-->TODO<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->TODO<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->TODO<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->TODO<!--/answer--> | <!--answer:function_warm_median_ms-->TODO<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
![[latency_comparison.png]]
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
For IAAS - our VM, nothing get's created - we install all necessary runtimes etc. prior to encapsulating the executable in container. The hard limit is the virtualized resources in the vm, fir e2-micro it is 2vCPU, 1 core, and 1 GB RAM. For PaaS Cloud Run, containers are auto spawned capped at hard limit --max-instances=3. For serverless cloud run functions, google provides the base image and necessary run-time. I didn't specify a cap on containers, but the default is 77.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
For IaaS Compute Engine, I first had to create the VM, do all of the provisioning steps, prepare install docker, authenticate with google artifact registry. Prior to this, I also had to setup a new VPN network, subnet, and ingress firewall rules matched with VM tag 'echo-http' to allow for http and ssh. For cloud run, I had to build the docker image and publish to google cloud, which however can be done with the command 'gcloud builds submit' in one go. In order to deploy the container, I ran 'gcloud run deploy echo-bot', contrary to cloud run functions where one merely specifies '--runtime' '--source' '--entrypoint' (and other flags) in order to deploy the function in 'gcloud functions deploy'.   
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
SERVICE   REGION    URL                                             LAST DEPLOYED BY                                              LAST DEPLOYED AT
✔  echo      us-east1  https://echo-141039911088.us-east1.run.app      service-141039911088@gcf-admin-robot.iam.gserviceaccount.com  2026-10-10T20:12:08.771217Z
✔  echo-bot  us-east1  https://echo-bot-141039911088.us-east1.run.app  jonas.jensen@eurecom.fr                                       2026-10-10T20:06:42.468940Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output of this command tells us that cloud run functions are deployed as cloud run services (containers) under the hood. This explains why the performance is so similar, and the primary difference whn it comes to deployment is that almost all abstraction are automized away from the developer in terms of specifying the necessary elements/resources in order to run aplication code. The essential information contained in the 'gcloud functions deploy' is the runtime, the location of our source code, and the actual application logic (the function). It is important to note here, that we in a declarative way tell google what kind of function it is by using the 'functions-framework' - in our case the method takes HTTP requests and returns a whatever is returned by the 'extract_message' method from app.py. In fact all source files in ./phase-1-echo-bot is 'copied' into the final container in order for the function to be able to run. W.r.t. the RTT being reported, it is actually rather the median of cloud run and functions that are quite close in my case, however the cold start RTTs are also relatively close.  
<!--/answer-->
