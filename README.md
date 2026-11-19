# Capstone Track: Networking and Cloud Infrastructure
 
**Final Sprint | Due before Final Presentations**

> **New to this track? Start here.** The shared [INET 4031 Capstone Guide](https://github.com/INET4031-Labs/inet4031-capstone-guide) explains how to work through any track on your own. Feeling lost in the first few days is normal for a capstone: read the guide's Start Here page, run the Environment Checkpoint below, and take the next small step.
>
> | I need... | Go to |
> |---|---|
> | A reading order and what to do in my first session | [START-HERE](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/START-HERE.md) |
> | What I may change in the app, and how to propose my own criterion | [MAKE-IT-YOURS](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/MAKE-IT-YOURS.md) |
> | Local VM: how my k3s and OpenTofu setup differs from this README | [LOCAL-VM](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/LOCAL-VM.md) |
> | What to prioritize in Weeks 11 to 14 | [WEEKLY-PLAN](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/WEEKLY-PLAN.md) and the Sprint Timeline below |
> | Plain-language explanations of this track's concepts | [CONCEPTS](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/CONCEPTS.md) |
> | A cheat sheet for each tool in this track | [TOOL-GUIDE](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/TOOL-GUIDE.md) |
> | Which file proves which success criterion | [EVIDENCE-CHECKLIST](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/EVIDENCE-CHECKLIST.md) |
> | How the Week 13 rebuild works | [REBUILD](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/REBUILD.md) and the Rebuild section below |
> | Help when something is broken | [GETTING-UNSTUCK](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/GETTING-UNSTUCK.md) |
> | How to use AI tools well on this project | [USING-AI](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/USING-AI.md) |
>
> **Local VM students:** much of this README describes the Shared VM setup (k3d, team numbers, `main`, `kubectl apply`). Your setup is a single-node k3s cluster that changes through OpenTofu and your deploy script. Look for the **Local VM** notes in each section, and keep [LOCAL-VM](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/LOCAL-VM.md) open while you work.
 
## Overview
 
Sam Whitfield, the Platform Team lead at Harborlight Logistics, has news. Harborlight just won a new depot contract, and Marcus Reyes (CTO) expects statustracker traffic to roughly triple once the new depot's dispatchers and drivers start filing incidents. Dana Okafor (Director of Operations) has promised the new customer that the Operations Support Desk will keep working through a bad afternoon. Marcus's instruction to the Platform Team was short: the platform has to scale up on its own and survive a node failure without anyone noticing.
 
Today the Flask, Nginx, and database Deployments each run a single replica. If the node that hosts the Flask pod goes away, the Support Desk sees errors until Kubernetes reschedules it. If load triples, nothing adds capacity. Sam wants you to find out what the cluster does today, design the changes that fix it, and prove they work with measurements, not opinions. Everything must run on the existing single server, with no new paid services, before the contract renewal in six weeks.
 
After completing this track, you will have a statustracker deployment that adds capacity under load, keeps serving traffic while a k3d node is stopped, and comes with measured evidence, a written network and service-discovery analysis, and a runbook another engineer can follow.
 
## The Business Problem
 
**Who is asking:** Sam Whitfield (technical customer) on behalf of Marcus Reyes (cost and release safety) and Dana Okafor (customer promises).
 
**Pain today:** One replica of each tier. No autoscaling. No guarantee that a node failure leaves anything running. Nobody on the team has written down how a request finds its way from the Nginx Service to a Flask pod, or which NetworkPolicies that path depends on.
 
**Constraints:**
- Budget: no new paid services. Everything runs on the existing single server shared by about six teams.
- Memory: scaling out costs memory. Your team has a hard budget (see Rules of the Road).
- Deadline: before the next quarterly business review.
- Safety: node-failure drills are done only on your own cluster's containers.
**Success criteria (measurable):**
1. Under a k6 load you design (at least 3x the request rate of your own recorded baseline), the Flask tier scales from its starting replica count to at least 2 replicas automatically. The `kubectl get hpa` output over time is saved in `capstone/networking-cloud/evidence/`.
2. With one k3d agent node stopped, k6 traffic to `http://localhost:<8090 + team number>/health` shows `http_req_failed` below 0.01 (under 1%) for a drill of at least 3 minutes. A drill file records the exact stop and start times.
3. A PodDisruptionBudget exists for each tier you scaled, and a file in `evidence/` shows what happens to those pods when a node is unavailable.
4. A written analysis in `ARCHITECTURE.md` traces one request from your k6 client to the database, naming every Service, DNS name, and NetworkPolicy involved, plus one NetworkPolicy you tightened or added with a before/after test.
5. The sum of memory limits of everything you added or scaled beyond today's baseline stays under the team budget, `maxReplicas` is capped in your autoscaler manifests, and the arithmetic is saved in a file.
How you get there is up to your team.
 
## Local VM Success Criteria

The criteria above assume the Shared VM's three-node k3d cluster with an Nginx tier. Your Local VM runs a single k3s node with `web` and `db` and no Nginx, so there is no agent node to stop. Criteria 1 and 5 apply as written, with `web` in place of Flask. Criteria 2 to 4 are replaced by these:

2. **(Local VM)** While k6 sends traffic to the `web` Service for at least 3 minutes, you (a) delete one `web` pod and (b) roll out a new `web` image tag through your pipeline (pull request, Deploy gate, `deploy.sh`). `http_req_failed` stays below 0.01 (under 1%) for the whole run. A drill file records the time of the deletion and the start and end of the rollout.
3. **(Local VM)** A PodDisruptionBudget for `web`, defined in `week-5/`, exists, and a file in `evidence/` shows `kubectl get pdb` and `kubectl describe pdb` at your starting replica count and when scaled, with a short written explanation of what the PDB does and does not protect against on a one-node cluster.
4. **(Local VM)** A written analysis in `ARCHITECTURE.md` traces one request from your k6 client to the database, naming every Service, DNS name, endpoint, and NetworkPolicy involved (including `db-clients`), plus one NetworkPolicy you added or tightened through `week-5/`, with a before/after test of both an allowed and a denied connection.

Your HPA, PDB, and any NetworkPolicy you add live in `week-5/` and reach the cluster through the Deploy gate, like everything else since Week 7. See [LOCAL-VM](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/LOCAL-VM.md).

## Learning Objectives
 
- Explain how Kubernetes Services and cluster DNS route a request to a pod, and verify it from the command line.
- Configure horizontal autoscaling and explain what it measures and why it lags.
- Use readiness and liveness probes, PodDisruptionBudgets, and scheduling spread to keep a service available during node loss.
- Refine NetworkPolicies without breaking the application, and test both the allowed and the denied path.
- Design and run a repeatable load test and a failure drill, and interpret the results.
- Describe infrastructure in OpenTofu and predict its effect without applying it (RHEL), or ship it through a gated OpenTofu pipeline (Local VM).
## Prerequisites
 
Everything from earlier weeks should work: the Compose stack, the 3-node k3d cluster with the Flask, Nginx, and database Deployments, the NetworkPolicies (default-deny plus allows), Prometheus in the `monitoring` namespace, and k6.

> **Local VM (Ubuntu, individual):** your prerequisites are Weeks 4 to 10 of your section: the k3s node, `web` and `db` managed by OpenTofu in `week-5/`, Prometheus (Week 6), the Deploy gate and `deploy.sh` (Week 7), the `db-clients` NetworkPolicy and resource limits (Weeks 8 and 9), and k6 with `week-10/run-k6.sh` (Week 10).
 
### Environment Checkpoint
 
Run each command and confirm the result before you plan anything. On the Shared VM, set `N` to your team number first (for example `export N=3`). On the Local VM, use the second block instead.
 
```bash
k3d cluster list
kubectl get nodes -o wide
kubectl get deploy,svc,pods -n default
kubectl get networkpolicy -n default
kubectl get pods -n monitoring
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:$((8090 + N))/health
kubectl top nodes
```
 
Expected: three nodes `Ready`, your Deployments available, the NetworkPolicies listed, Prometheus pods Running, and `200` from the health check. If `kubectl top` errors, tell the TA. To see the container names of your own nodes:
 
```bash
docker ps --filter name=k3d-<your cluster name>-
```

**Local VM:**

```bash
kubectl get nodes -o wide
kubectl get deploy,svc,pods -n default
kubectl get networkpolicy -n default
kubectl get pods -n monitoring
curl -s -o /dev/null -w "%{http_code}\n" http://$(kubectl get service web -o jsonpath='{.spec.clusterIP}'):8080/health
kubectl top nodes
tail -5 ~/deploy.log
```

Expected on the Local VM: one node `Ready`, `web` and `db` available, `db-clients` listed, Prometheus pods Running, `200` from the health check, and a recent `up to date` line in the deploy log.
 
## Team Port Assignments
 
This track needs no new ports. Use the ports you already have:
 
- Compose app: 8080 + team number
- Kubernetes (k3d load balancer): 8090 + team number (8091-8097 for teams 1-7)
- Prometheus port-forward: 9090 + team number
> **Shared VM (RHEL, teams):** Your Kubernetes port is 8090 + your team number. Run k6 against that port only. Never send traffic to another team's port.
 
> **Local VM (Ubuntu, individual):** There is no team number and no k3d load balancer. Run k6 on the VM against the `web` Service address, the way `./week-10/run-k6.sh` does (`http://$(kubectl get service web -o jsonpath='{.spec.clusterIP}'):8080`); the Service spreads requests across every `web` pod, which a port-forward does not. For your browser, `kubectl port-forward --address 0.0.0.0 service/web 8080:8080` and open http://localhost:8080. For Prometheus, `kubectl port-forward --namespace monitoring service/prometheus-server 9090:80` and `./week-6/pq.sh` (your Week 6 setup).
 
## Tools Available
 
| Tool | What it's for | Verify (both OSs) | Ubuntu install |
|------|---------------|-------------------|----------------|
| [kubectl](https://kubernetes.io/docs/reference/kubectl/) | Inspect and change cluster objects, `kubectl top` | `kubectl version --client` | Already installed in earlier weeks (keep the one you have; it must stay within one minor version of your cluster) |
| [k3d](https://k3d.io/) | Cluster info, image import | `k3d version` | Already installed in earlier weeks |
| [k6](https://grafana.com/docs/k6/latest/) | Load tests and failure-drill traffic | `k6 version` | Download the latest release (see below) |
| [helm](https://helm.sh/docs/) | Inspect the Prometheus release | `helm version --short` | Already installed in earlier weeks |
| [docker](https://docs.docker.com/) | Stop and start your own k3d node containers | `docker ps --filter name=k3d-` | Already installed in earlier weeks |
| [OpenTofu](https://opentofu.org/docs/) | Write infrastructure definitions (RHEL: write the code only, never run `tofu`) | Local VM only: `tofu version` (on RHEL, do not run any `tofu` command) | Download the latest release (see below) |
| [jq](https://jqlang.org/manual/) | Read JSON from `kubectl -o json` and k6 summaries | `jq --version` | `sudo apt-get install -y jq` |
| [curl](https://curl.se/docs/) and [dig](https://bind9.readthedocs.io/en/latest/manpages.html) | Quick request and DNS checks from the VM | `curl --version` and `dig -v` | `sudo apt-get install -y curl dnsutils` |
 
> **Shared VM (RHEL, teams):** The system tools in this table (everything except Python packages in your own virtualenv) are pre-installed by the TA. Run the Verify command for each tool. If a command is not found, post in the course channel or tell the TA or professor before continuing. Do not install anything system-wide.
 
> **Local VM (Ubuntu, individual):** Your VM runs Ubuntu Server 26.04.1. You install tools yourself. Run the command in the last column, then the Verify command. k3d is not part of your setup: skip its row and record `kubectl get nodes -o wide` instead. kubectl came with k3s (Week 4), OpenTofu with Week 5, Helm with Week 6, and k6 with Week 10, so if their Verify commands work, skip the downloads below.
 
**Versions.** This table has no version numbers on purpose: use the current stable release of each tool, so these instructions keep working in future semesters. Record what you actually used. Run each Verify command and save the output to `capstone/networking-cloud/TOOL-VERSIONS.txt`, then commit it.
 
**Single-binary installs (Ubuntu only: k6 and OpenTofu).** Define the helper below (paste it into your terminal), then download the newest release:
 
```bash
# Print the newest release tag of a GitHub project, for example: latest_tag aquasecurity/trivy
# It follows the /releases/latest redirect, so it needs no login and no API key.
latest_tag() { curl -fsSLI -o /dev/null -w '%{url_effective}\n' "https://github.com/$1/releases/latest" | sed 's|.*/tag/||'; }
```
 
```bash
cd "$(mktemp -d)"
# k6
V=$(latest_tag grafana/k6)
curl -fLO "https://github.com/grafana/k6/releases/download/${V}/k6-${V}-linux-amd64.tar.gz"
tar -xzf "k6-${V}-linux-amd64.tar.gz"
sudo install -m 0755 "k6-${V}-linux-amd64/k6" /usr/local/bin/
# OpenTofu
V=$(latest_tag opentofu/opentofu)
curl -fLO "https://github.com/opentofu/opentofu/releases/download/${V}/tofu_${V#v}_linux_amd64.zip"
unzip -o "tofu_${V#v}_linux_amd64.zip" tofu
sudo install -m 0755 tofu /usr/local/bin/
```
 
If a download returns "404 Not Found", the project renamed its release file: open its releases page, copy the exact file name, and adjust the command. Compare the download with the checksum file on the release page when one is published.
 
No Python packages are required. If you write helper scripts in Python, create a virtualenv in your home directory (`python3 -m venv ~/networking-cloud-venv`); never use system `pip`.
 
> **Background:** metrics-server ships inside k3s, and therefore in the k3d node image. It feeds `kubectl top` and is what a HorizontalPodAutoscaler reads. You do not install it. If `kubectl top` errors, tell the TA.
 
## Example Approaches
 
These are examples, not requirements. Combine them, change them, or ignore them.
 
**Approach A: Autoscale-first.** Give the Flask Deployment honest requests and limits, add a HorizontalPodAutoscaler with a capped `maxReplicas`, and drive it with a stepped k6 profile. This shows how a CPU target, the metrics interval, and pod start time determine how quickly capacity arrives, and what users feel during the gap.
 
**Approach B: Survive-the-failure-first.** Run two or more replicas of each stateless tier, spread them across nodes with scheduling rules, protect them with PodDisruptionBudgets, and tune readiness probes so a dead pod leaves the Service quickly. Prove it with a `docker stop` drill on one of your own agent nodes while k6 runs. This shows the difference between "restarts eventually" and "keeps serving".
 
> **Local VM:** in Approach B, the drill is the pod-failure and rollout drill from the Local VM Success Criteria, and spreading replicas across nodes does not apply on one node. In Approach C, your OpenTofu is real, not [PREDICT ONLY].

**Approach C: Understand-the-network-first.** Document how names resolve and how a request is routed (Service, endpoints, kube-proxy, CoreDNS), then refine NetworkPolicies so the allowed path is exactly what the scaled-out design needs, testing both allowed and denied connections. Add an OpenTofu description of your scaling objects as **[PREDICT ONLY]** on RHEL. This shows that scaling changes the network picture, not only the pod count.
 
## Track Tips and Common Trip-Ups

These notes come from where students usually get stuck. They do not change the success criteria; they help you reach them. Concepts are explained in [CONCEPTS](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/CONCEPTS.md) and commands in [TOOL-GUIDE](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/TOOL-GUIDE.md). If a flag here does not work on your installed version, check the tool's `--help`.

**Designing your k6 tests (criteria 1 and 2)**

- **Measure the baseline rate first.** Run a short, steady test (a few virtual users for a minute or two) before changing anything. The k6 summary's `http_reqs` line gives requests per second, and `http_req_duration` gives p95. That rate is the "1x" that criterion 1 multiplies by three.
- **Control the rate, not just the users.** With a fixed number of virtual users, the request rate drops when the app slows down, so "3x" drifts. k6's arrival-rate executors (`constant-arrival-rate` and `ramping-arrival-rate`) hold a request rate you choose, which makes 3x easy to prove. Read the k6 scenarios documentation for your version.
- **Step, then hold.** A stepped profile (1x, then 2x, then 3x) with each step held for a few minutes gives the HPA time to react and gives you a clear before and after in `hpa-over-time.txt`.
- **Put the pass/fail in the script.** A threshold such as `http_req_failed: ['rate<0.01']` makes the drill report its own result, so criterion 2's evidence is in the k6 summary.
- **Commit every script** under `capstone/networking-cloud/k6/` with a comment saying what it tests. On the Shared VM, ramp gently: six teams share the CPU, and an overloaded VM measures your neighbors as much as your app.

**Autoscaling (criterion 1)**

- An HPA measures CPU as a percentage of the pods' CPU **request**. If the Deployment has no CPU request, `kubectl get hpa` shows `<unknown>` and nothing scales. `kubectl describe hpa <name>` explains why.
- Scale-up takes time (metric interval plus pod start), and scale-down waits several minutes by default. Make your k6 run long enough to see the change and record over time, for example `kubectl get hpa -w | tee capstone/networking-cloud/evidence/hpa-over-time.txt` in its own terminal while k6 runs.
- Memory arithmetic, as an example: if each Flask pod has a 128Mi limit and the HPA can go from 1 to 4 replicas, the worst case adds 3 x 128Mi = 384Mi. Add every other addition and compare the total to the 768Mi budget.

**Surviving a node failure (criteria 2 and 3)**

- A PodDisruptionBudget limits **voluntary** disruptions, such as a node drain. `docker stop` on a node is an **involuntary** failure, so a PDB will not keep pods alive there. What keeps traffic flowing is having replicas on other nodes plus readiness probes. Criterion 3 asks you to show what actually happened, so record it honestly.
- Before a drill, run `kubectl get pods -o wide` and note which node each pod is on. The database is not scaled; if it shares the node you stop, decide whether that drill measures what you intended, and write down which node you chose and why.
- When a node stops, Kubernetes first marks it `NotReady`, which removes its pods from Service endpoints, and only replaces the pods later (by default after several minutes). Timestamp both moments; discussion question 3 asks for them.

**Tracing a request (criterion 4)**

Fill in a path like this one and name every object. Do not guess; find each hop with a command.

| Hop | Object (name) | How you found it |
|---|---|---|
| k6 on the VM | `http://localhost:<8090 + N>` | your k6 script |
| k3d load balancer container | | `docker ps --filter name=k3d-<cluster>-` |
| Nginx Service and pods | | `kubectl get svc,endpoints` |
| Flask Service, DNS name, and pods | | `kubectl describe svc <name>`; DNS lookup from a debug pod |
| Database Service and pod | | |
| NetworkPolicies allowing each hop | | `kubectl get networkpolicy -o yaml` |

A debug pod needs a memory limit, which `kubectl run` cannot set in current versions. Use a small manifest and delete the pod afterward:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-debug
  labels:
    app: dns-debug
spec:
  restartPolicy: Never
  containers:
    - name: debug
      image: busybox
      command: ["sleep", "600"]
      resources:
        requests:
          memory: "16Mi"
        limits:
          memory: "64Mi"
```

Then `kubectl exec dns-debug -- nslookup <service name>` and `kubectl delete pod dns-debug`. Because its label matches no allow policy, this pod is also a ready-made "denied" client for your NetworkPolicy before and after test.

**Local VM notes (criteria 1 to 5)**

- **Tell OpenTofu to leave the replica count alone.** Once an HPA manages `web`, the `replicas` line in `web.tf` and the HPA disagree, and every deploy resets the count. Add `lifecycle { ignore_changes = [spec[0].replicas] }` to the `web` Deployment resource, and read the provider documentation for your version to see why.
- **Your `web` pods already have a CPU request** (Week 8), so the HPA has something to measure. Check `kubectl describe hpa` before assuming it works.
- **A rollout without failures needs more than one replica.** With one pod, the old pod can stop before the new one is ready. Look at the Deployment's rolling update settings (`max_unavailable`, `max_surge`) and your readiness probe timing, and decide what your drill needs.
- **Deleting a pod is an involuntary disruption.** The PDB does not stop `kubectl delete pod`. It limits evictions, such as a drain. On one node, draining also cordons the only node, so write down what that would mean before you try anything.
- **Import the new image before the pull request.** The rollout in criterion 2 is a real deploy. If k3s does not have the new tag when `deploy.sh` applies it, the new pods cannot start, which is a failed drill.
- **Trace these hops:** k6 on the VM, the `web` Service address (kube-proxy), its endpoints, a `web` pod, the `db` DNS name, the `db` Service and its endpoint, the `db` pod, and the `db-clients` policy. There is no load balancer container and no Nginx. Port-forwards bypass NetworkPolicy, so test allowed and denied paths from debug pods, and say in your analysis whether k6 running on the VM itself is affected by the policy you added.

## Rules of the Road
 
- **Own cluster and ports only.** Never scan, probe, or load-test anything outside your team's cluster and app. No nmap of the VM or campus network. k6 targets `http://localhost:<8090 + team number>` and nothing else.
- **Node drills touch only your own containers (Shared VM).** The Docker daemon is shared by all teams. Before any drill run `docker ps --filter name=k3d-<your cluster name>-` and confirm every name starts with your cluster name. Stop and start only those, for example `docker stop k3d-<your cluster name>-agent-0`, and always `docker start` it again when the drill ends. Never stop the server (control-plane) node, the load balancer container, or any container not matching your name. If a node does not return to `Ready` within a few minutes, tell the TA.
- **Memory budget.** Keep the memory limits of everything you add or scale (extra replicas, debug pods) under **768Mi total** beyond today's baseline. Cap `maxReplicas` at 4 for Flask and 3 for Nginx. Do not scale the database. Every pod you deploy must have memory `requests` and `limits`. Check with `kubectl top pods -A` and `free -h`. If `kubectl top` errors, tell the TA, and meanwhile use `kubectl describe` requests/limits plus `free -h`. Keep load tests to a few minutes and ramp gently; about six teams share the CPU.
- **Debug pods.** If you use a temporary pod for DNS checks, give it a memory limit of 64Mi or less, remember that the default-deny policy applies to traffic into it, and delete it when you finish (`kubectl delete pod <name>`). Do not leave it running.
- **New pods need NetworkPolicy allows.** The default-deny-ingress policy blocks any new pod that receives traffic until you write an allow policy for it. Extra replicas of existing Deployments match the existing label-based allows; a new kind of pod does not.
- **Builds.** If you change an image, build with `docker build --network host -t <image>:team<N> <dir>` and import with `k3d image import <image>:team<N> -c <your cluster name>`. Because of `umask 077`, edited files become mode 600; in a Dockerfile with a non-root user use `COPY --chmod=644` (or `chmod 644` before building).
- **Port-forwards.** Run `kubectl port-forward` in the foreground in its own terminal. Never background it with `&`. To clean up: `pkill -u $USER -f "kubectl port-forward"`.
- **No sudo.** Run k3d, kubectl, helm, and k6 without `sudo`. Students never use sudo on RHEL.
- **OpenTofu.** On RHEL, do not run any `tofu` command (it is installed, but all teams share one VM and it would touch shared state). Do not run `tofu plan` or `tofu apply`. Write the code, mark it **[PREDICT ONLY]**, write your prediction in the Google Doc, and apply the real change live with `kubectl`. On the Local VM, OpenTofu is how your changes ship: see the Local VM rules below.
- **Git on RHEL.** `git switch -c <branch>`, `git push -u origin <branch>`, open a PR, wait for build-and-push, then merge. Never push to main.
- Never disable firewalld or edit `/etc/docker/daemon.json`.

### Local VM (Ubuntu, individual) rules

The rules above that mention other teams, team numbers, k3d, `--network host`, firewalld, `/etc/docker/daemon.json`, `umask 077`, or [PREDICT ONLY] describe the Shared VM. On your own VM these apply instead:

- **Change the cluster through OpenTofu.** Anything that should last goes in `week-5/*.tf`, through a pull request, the Deploy gate, and `week-7/deploy.sh`. Throwaway test pods may use `kubectl apply`, and you delete them afterward. A change made with `kubectl` to an object OpenTofu manages is undone by the next deploy.
- **The gate checks your additions.** Every container you add in `week-5/` needs CPU and memory requests and limits and must not allow privilege escalation, or the `security scan` job fails and the ruleset blocks the merge.
- **New image, new tag.** Build a new tag (for example `:v1`, `:v2`), import it with `docker save` and `sudo k3s ctr images import` **before** the pull request that uses it, and never overwrite the tag that is running.
- **`master` and `protect-master` stay.** Work on branches and merge through pull requests. Do not delete the ruleset, remove its checks, or add a bypass.
- **No default-deny.** Only the `db` pods are protected (by `db-clients`). A pod you add accepts traffic from anywhere until a NetworkPolicy selects it.
- **sudo** is for package installs, `sudo install` of single binaries, and `sudo k3s ctr`. Run `kubectl`, `helm`, `tofu`, and `k6` as your normal user.
- **Memory budget still applies:** 768Mi beyond your baseline, `maxReplicas` at most 4 for `web`, and the database is not scaled. Check `free -h` before and after.
- **Drills:** delete only `web` pods, one at a time. Never delete the `db` pod during a drill.

## How to Run This Sprint
 
### Sprint Timeline (Weeks 11 to 14)

Both sections follow the same four weeks. The planning steps below still apply; this table says where your work should be at the end of each week. The full plan is in [WEEKLY-PLAN](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/WEEKLY-PLAN.md).

| Week | By the end of the week |
|---|---|
| 11 Kickoff | Environment Checkpoint saved. Baseline recorded before any change (request rate, p95, replica count, memory). Shared VM: `docker ps --filter name=k3d-<cluster>-` output saved. Local VM: `kubectl get nodes -o wide` and `kubectl get pods -o wide` saved. Request path sketched. QA test cases drafted. App changes decided ([MAKE-IT-YOURS](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/MAKE-IT-YOURS.md)). |
| 12 Build | Requests and limits, probes, PodDisruptionBudgets, and a capped HPA in place (Local VM: through `week-5/` and the Deploy gate). Shared VM: replicas spread across nodes. First scale-up captured. One practice drill run (Local VM: the pod-failure and rollout drill). Memory arithmetic ready for the mid-sprint check-in. |
| 13 Consolidate and rebuild | Final 3-minute drill evidence, NetworkPolicy before and after test, request trace in `ARCHITECTURE.md`, RUNBOOK, rebuild, and gap log. |
| 14 Present | Rebuild presentation (live on the Shared VM, recorded on the Local VM) plus your scale-up and drill evidence. |

### Shared VM (RHEL, teams)
 
1. **Sprint planning.** Turn each success criterion into GitHub issues on your project board, including separate issues for baseline measurement, autoscaling, the failure drill, the network analysis, and documentation.
2. **Roles (suggested).**
   - Scrum Master: runs standups, keeps the board current, owns the retrospective.
   - System Admin: owns cluster changes, node drills, and the memory budget.
   - Developers: write manifests, probes, policies, and k6 scripts.
   - QA: writes test cases in `docs/qa-report-final.md` **before** the implementation is done, then runs them.
3. **Baseline first.** Record request rate, p95, replica count, and memory before changing anything. Every later claim compares against it.
4. **Mid-sprint check-in.** With the TA or professor: show the baseline, your design sketch, and your memory budget arithmetic.
5. **Retrospective** at the end in `docs/final-sprint-retrospective.md`.
### Local VM (Ubuntu, individual)
 
You follow the same Sprint Timeline as the Shared VM teams, alone, using the Local VM Success Criteria.

- **Week 11:** Environment Checkpoint saved, baseline recorded before any change, request path sketched, and your own test cases drafted in `docs/qa-report-final.md`.
- **Week 12:** HPA and PodDisruptionBudget added through `week-5/` and the Deploy gate, rollout settings and probes checked, first scale-up captured, one practice drill, memory arithmetic ready for your check-in.
- **Week 13:** final 3-minute drill evidence, NetworkPolicy before and after test, request trace in `ARCHITECTURE.md`, RUNBOOK, then rebuild on a new VirtualBox VM and keep a gap log (see the Rebuild section).
- **Week 14:** recorded presentation of your rebuild.
- A short reflection replaces the retrospective.

## Rebuild (Weeks 13 and 14)

Your work has to come back up from your repository alone. Start from a clean VM, clone your repo, and follow only your README, RUNBOOK, and scripts. Every step that fails or that your README forgot is a gap: write it in `capstone/networking-cloud/REBUILD-GAPS.md`, fix it by pull request, and update the README. The full contract, the "what lives outside git" checklist, and the gap log template are in [REBUILD](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/REBUILD.md).

> **Shared VM (RHEL, teams):** In Week 13 your team takes a turn in a queue to test the rebuild on a separate VM, which is wiped between teams. In Week 14 you present live: the tools are pre-installed, and your team clones the repo and brings the app up from your README and setup instructions. There is no sudo on the rebuild or presentation VM, so every step your README asks someone to run must work without it. Your Ansible playbooks and any [PREDICT ONLY] code stay reference-only: do not run them, and make sure bringing the app up does not depend on them. Recreate your k3d cluster with the same `--subnet 100.97.<team>.0/24` flag you used before.

> **Local VM (Ubuntu, individual):** In Week 13 create a new VM in VirtualBox using the same steps you used for your original VM, clone your repo, and follow your README. Keep the original VM until the rebuild works. In Week 14 you present a recording of the rebuild. Start from the "Rebuild on a fresh VM" sections you wrote in your Week 7, 8, and 9 READMEs (k3s, OpenTofu variables, image import, deploy and backup cron lines), then add everything this track added. The full Local VM order is in [REBUILD](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/REBUILD.md). You have sudo on your own VM; every step that needs it must be in your README.

**Proof of life for this track:** three nodes `Ready` (Local VM: one node); your HPA and PodDisruptionBudgets exist; `/health` returns 200 on your Kubernetes port (Local VM: on the `web` Service address); a 60-second k6 run completes.

**Gaps this track usually finds:**
- The `k3d cluster create` command (with every flag) and the Prometheus helm install were typed by hand in earlier weeks and never committed.
- k6 scripts or HPA and PDB manifests applied from a local file that was never pushed.
- The baseline numbers live only in someone's terminal history.
- Local VM: the HPA or PDB was applied with `kubectl` during testing and never added to `week-5/`, so the new VM does not have it.
- Local VM: the new `web` image tag was imported on the old VM only, so the first deploy on the new VM cannot start the pods.

## Deliverables
 
- Implementation committed under `capstone/networking-cloud/` in your repo (merged via PR on the RHEL section)
- `capstone/networking-cloud/RUNBOOK.md`: how to operate, check, and recover the implementation
- `capstone/networking-cloud/TOOL-VERSIONS.txt`: the tool versions you actually used (see Tools Available)
- `capstone/networking-cloud/ARCHITECTURE.md` with an architecture diagram (Mermaid OK) showing VM, cluster, app, and new components
- A "How to run this" section added to the repo's top-level README so someone who clones the repo can bring the track implementation up
- `docs/qa-report-final.md` with your own test cases and results
- `capstone/networking-cloud/REBUILD-GAPS.md`: the gap log from your Week 13 rebuild (template in [REBUILD](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/REBUILD.md))
- Shared VM: `docs/final-sprint-retrospective.md`; Local VM: a short reflection (a section in the retrospective file named "Individual Reflection")
- Final presentation/demo: 8-10 minutes for a team on RHEL (CLI-driven: baseline, scale-up under load, node-failure drill with k6 running, network trace); 5 minutes for an individual on Local VM (baseline, scale-up, the pod-failure and rollout drill, network trace)
- Week 14 rebuild presentation: live on the Shared VM (clone the repo and bring the app up from your README), recorded on the Local VM. See the Rebuild section.
- Track-specific: `capstone/networking-cloud/evidence/` with saved command output for each success criterion (HPA over time, k6 summaries, drill start and stop times, memory budget arithmetic)
- Track-specific: the network and service-discovery analysis in `ARCHITECTURE.md`, including one before/after NetworkPolicy test
- Track-specific: an OpenTofu description of your scaling objects marked **[PREDICT ONLY]** on RHEL, with your written prediction in the Google Doc. Local VM: the HPA, PDB, and any NetworkPolicy you add live in `week-5/` and were applied by your pipeline.
**Save the output to a file and commit it** for each piece of evidence (or screenshot your terminal and add it to the Google Doc).
 
## Discussion (add to Google Doc)
 
1. Traffic is expected to triple. Which part of your system hits its limit first, and how do you know? What did you measure to prove it?
2. An autoscaler reacts after load arrives. What does a user experience in the gap, and what would you change (replica floor, probes, request sizing) to shrink it? What does that cost in memory?
3. During your node drill, how long did the cluster take to notice the failure and stop sending traffic to the lost pods? Which settings control that time? (Local VM: during your pod-failure drill, how long did it take for the deleted pod to leave the Service and for its replacement to pass readiness?)
4. Scaling the database was not on the table. Why is the database tier harder to make highly available than Flask, and what would Harborlight need to decide before investing?
5. Marcus Reyes cares about cost. Using your measured memory numbers, what is the cheapest design that still meets Dana Okafor's promise to the new customer?
## Getting Help
 
Clarifications go to the TA or professor. There is no wiki for the capstone tracks.

**Before you ask:** work through [GETTING-UNSTUCK](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/GETTING-UNSTUCK.md) for about 30 minutes, then ask with the command, its full output, what you expected, and what you tried.

**Using AI:** AI assistants are welcome as a tool for learning how these tools work and how enterprises use them. Do not rely on them to produce your work: understand every line you commit, verify AI answers against the tool's documentation, never paste AI output into evidence files, and never paste secrets into an AI tool. See [USING-AI](https://github.com/INET4031-Labs/inet4031-capstone-guide/blob/main/USING-AI.md).
