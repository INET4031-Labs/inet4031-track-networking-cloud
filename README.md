# Capstone Track: Networking and Cloud Infrastructure

**Final Sprint | Due before Final Presentations**

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

## Learning Objectives

- Explain how Kubernetes Services and cluster DNS route a request to a pod, and verify it from the command line.
- Configure horizontal autoscaling and explain what it measures and why it lags.
- Use readiness and liveness probes, PodDisruptionBudgets, and scheduling spread to keep a service available during node loss.
- Refine NetworkPolicies without breaking the application, and test both the allowed and the denied path.
- Design and run a repeatable load test and a failure drill, and interpret the results.
- Describe infrastructure in OpenTofu and predict its effect without applying it (RHEL).

## Prerequisites

Weeks 2 through 9 should work: the Compose stack, the 3-node k3d cluster with the Flask, Nginx, and database Deployments, the NetworkPolicies (default-deny plus allows), Prometheus in the `monitoring` namespace, and k6.

### Environment Checkpoint

Run each command and confirm the result before you plan anything. On the Shared VM, set `N` to your team number first (for example `export N=3`). On the Local VM there is no team number: replace `$((8090 + N))` with `8081` in the `curl` line.

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

## Team Port Assignments

This track needs no new ports. Use the ports you already have:

- Compose app: 8080 + team number
- Kubernetes (k3d load balancer): 8090 + team number (8091-8097 for teams 1-7)
- Prometheus port-forward: 9090 + team number

> **Shared VM (RHEL, teams):** Your Kubernetes port is 8090 + your team number. Run k6 against that port only. Never send traffic to another team's port.

> **Local VM (Ubuntu, individual):** There is no team number. Use http://localhost:8081 (forwarded by VS Code) for the app, and your own Prometheus port-forward as in Week 5.

## Tools Available

| Tool | What it's for | Verify (both OSs) | Ubuntu install |
|------|---------------|-------------------|----------------|
| kubectl | Inspect and change cluster objects, `kubectl top` | `kubectl version --client` | Already installed in earlier weeks (keep the one you have; it must stay within one minor version of your cluster) |
| k3d | Cluster info, image import | `k3d version` | Already installed in earlier weeks |
| k6 | Load tests and failure-drill traffic | `k6 version` | Download the latest release (see below) |
| helm | Inspect the Prometheus release | `helm version --short` | Already installed in earlier weeks |
| docker | Stop and start your own k3d node containers | `docker ps --filter name=k3d-` | Already installed in earlier weeks |
| OpenTofu | Write infrastructure definitions (RHEL: write the code only, never run `tofu`) | Local VM only: `tofu version` (on RHEL, do not run any `tofu` command) | Download the latest release (see below) |
| jq | Read JSON from `kubectl -o json` and k6 summaries | `jq --version` | `sudo apt-get install -y jq` |
| curl and dig | Quick request and DNS checks from the VM | `curl --version` and `dig -v` | `sudo apt-get install -y curl dnsutils` |

> **Shared VM (RHEL, teams):** The system tools in this table (everything except Python packages in your own virtualenv) are pre-installed by the TA. Run the Verify command for each tool. If a command is not found, post in the course channel or tell the TA or professor before continuing. Do not install anything system-wide.

> **Local VM (Ubuntu, individual):** Your VM runs Ubuntu Server 26.04.1. You install tools yourself. Run the command in the last column, then the Verify command.

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

**Approach C: Understand-the-network-first.** Document how names resolve and how a request is routed (Service, endpoints, kube-proxy, CoreDNS), then refine NetworkPolicies so the allowed path is exactly what the scaled-out design needs, testing both allowed and denied connections. Add an OpenTofu description of your scaling objects as **[PREDICT ONLY]** on RHEL. This shows that scaling changes the network picture, not only the pod count.

## Rules of the Road

- **Own cluster and ports only.** Never scan, probe, or load-test anything outside your team's cluster and app. No nmap of the VM or campus network. k6 targets `http://localhost:<8090 + team number>` and nothing else.
- **Node drills touch only your own containers.** The Docker daemon is shared by all teams. Before any drill run `docker ps --filter name=k3d-<your cluster name>-` and confirm every name starts with your cluster name. Stop and start only those, for example `docker stop k3d-<your cluster name>-agent-0`, and always `docker start` it again when the drill ends. Never stop the server (control-plane) node, the load balancer container, or any container not matching your name. If a node does not return to `Ready` within a few minutes, tell the TA.
- **Memory budget.** Keep the memory limits of everything you add or scale (extra replicas, debug pods) under **768Mi total** beyond today's baseline. Cap `maxReplicas` at 4 for Flask and 3 for Nginx. Do not scale the database. Every pod you deploy must have memory `requests` and `limits`. Check with `kubectl top pods -A` and `free -h`. If `kubectl top` errors, tell the TA, and meanwhile use `kubectl describe` requests/limits plus `free -h`. Keep load tests to a few minutes and ramp gently; about six teams share the CPU.
- **Debug pods.** If you use a temporary pod for DNS checks, give it a memory limit of 64Mi or less, remember that the default-deny policy applies to traffic into it, and delete it when you finish (`kubectl delete pod <name>`). Do not leave it running.
- **New pods need NetworkPolicy allows.** The default-deny-ingress policy blocks any new pod that receives traffic until you write an allow policy for it. Extra replicas of existing Deployments match the existing label-based allows; a new kind of pod does not.
- **Builds.** If you change an image, build with `docker build --network host -t <image>:team<N> <dir>` and import with `k3d image import <image>:team<N> -c <your cluster name>`. Because of `umask 077`, edited files become mode 600; in a Dockerfile with a non-root user use `COPY --chmod=644` (or `chmod 644` before building).
- **Port-forwards.** Run `kubectl port-forward` in the foreground in its own terminal. Never background it with `&`. To clean up: `pkill -u $USER -f "kubectl port-forward"`.
- **No sudo.** Run k3d, kubectl, helm, and k6 without `sudo`. Students never use sudo on RHEL.
- **OpenTofu.** On RHEL, do not run any `tofu` command (it is installed, but all teams share one VM and it would touch shared state). Do not run `tofu plan` or `tofu apply`. Write the code, mark it **[PREDICT ONLY]**, write your prediction in the Google Doc, and apply the real change live with `kubectl`. On the Local VM you may run them.
- **Git on RHEL.** `git switch -c <branch>`, `git push -u origin <branch>`, open a PR, wait for build-and-push, then merge. Never push to main.
- Never disable firewalld or edit `/etc/docker/daemon.json`.

## How to Run This Sprint

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

- **Week 1:** baseline measurements, network and DNS analysis, probes and replicas, and your own test cases in `docs/qa-report-final.md`.
- **Week 2:** autoscaling, failure drill, tightened policy, runbook, architecture diagram, and presentation. A short reflection replaces the retrospective.

## Deliverables

- Implementation committed under `capstone/networking-cloud/` in your repo (merged via PR on the RHEL section)
- `capstone/networking-cloud/RUNBOOK.md`: how to operate, check, and recover the implementation
- `capstone/networking-cloud/TOOL-VERSIONS.txt`: the tool versions you actually used (see Tools Available)
- `capstone/networking-cloud/ARCHITECTURE.md` with an architecture diagram (Mermaid OK) showing VM, cluster, app, and new components
- A "How to run this" section added to the repo's top-level README so someone who clones the repo can bring the track implementation up
- `docs/qa-report-final.md` with your own test cases and results
- Shared VM: `docs/final-sprint-retrospective.md`; Local VM: a short reflection (a section in the retrospective file named "Individual Reflection")
- Final presentation/demo: 8-10 minutes for a team on RHEL (CLI-driven: baseline, scale-up under load, node-failure drill with k6 running, network trace); 5 minutes for an individual on Local VM
- Track-specific: `capstone/networking-cloud/evidence/` with saved command output for each success criterion (HPA over time, k6 summaries, drill start and stop times, memory budget arithmetic)
- Track-specific: the network and service-discovery analysis in `ARCHITECTURE.md`, including one before/after NetworkPolicy test
- Track-specific: an OpenTofu description of your scaling objects marked **[PREDICT ONLY]** on RHEL, with your written prediction in the Google Doc

**Save the output to a file and commit it** for each piece of evidence (or screenshot your terminal and add it to the Google Doc).

## Discussion (add to Google Doc)

1. Traffic is expected to triple. Which part of your system hits its limit first, and how do you know? What did you measure to prove it?
2. An autoscaler reacts after load arrives. What does a user experience in the gap, and what would you change (replica floor, probes, request sizing) to shrink it? What does that cost in memory?
3. During your node drill, how long did the cluster take to notice the failure and stop sending traffic to the lost pods? Which settings control that time?
4. Scaling the database was not on the table. Why is the database tier harder to make highly available than Flask, and what would Harborlight need to decide before investing?
5. Marcus Reyes cares about cost. Using your measured memory numbers, what is the cheapest design that still meets Dana Okafor's promise to the new customer?

## Getting Help

Clarifications go to the TA or professor. There is no wiki for the capstone tracks.
