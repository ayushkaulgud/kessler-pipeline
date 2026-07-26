# kessler-pipeline

Distributed pipeline that ingests public orbital catalog data and screens roughly 30k objects for close approaches. Runs on Kubernetes. Stores history bitemporally (valid time and transaction time) so any screening run can be replayed as of a past moment.

**Status: in progress.** Design document below. Roadmap at the bottom shows the project plan through the upcoming semester.

## Objective

This is a side project I am building through the academic semester, and the primary goal is to learn the parts of the stack I have not worked in before: Linux and networking fundamentals, distributed systems, containers and Kubernetes, and production data modeling. The orbital mechanics is the part I already know, so it is the domain I chose to learn the rest through rather than the point of the exercise.

Scope is set accordingly. It is built in phases, each phase is meant to teach something specific, and design decisions get written down here with the alternatives I rejected so the reasoning is checkable later, including by me.

## Problem

Orbits are deterministic. Knowing where things are is not.

Public TLE data is stale, low precision, irregularly updated, and carries no published covariance. The output that matters is the uncertainty around a predicted approach, not the miss distance alone.

## Prior work

Conjunction screening is a solved problem and this is not a new service.

- CelesTrak SOCRATES, running since 2004, publishes conjunction reports twice daily
- NASA CARA Analysis Tools, open sourced by the Goddard conjunction assessment team. I spoke with them briefly during my internship at Goddard.
- SatGuard, MIT licensed Python pipeline with SGP4, Foster/Chan/Alfano Pc, and CDM output
- Orekit, Java flight dynamics library with collision probability support
- AstriaGraph (ut-astria), UT Austin platform for tracking resident space objects. I spoke with Dr. Moriba Jah, who leads the group behind it, at UT Austin.

A full catalog screening pass is reportedly achievable in seconds on a single machine using a KD-tree. So the distributed design here is not justified by one pass. It is justified by replay: re-screening historical windows as of every past epoch, sweeping thresholds, and serving concurrent point in time queries. Phase 2 benchmarks will confirm or kill that assumption.

## What it does

1. Pulls TLEs on a schedule and keeps every version ever seen
2. Pulls upcoming launch events and stores them as structured records
3. Propagates the catalog with SGP4 and screens for close approaches
4. Reports probability of collision from propagated covariance
5. Replays any past screening run using only the data that existed at that moment

## What it does not do

- No high precision orbit determination. SGP4 from public TLEs, kilometer scale errors.
- No operational use. This is not a collision avoidance service.
- No prediction of launch trajectories. Scheduled launches enter as events, not as propagated objects.
- No real time alerting. The unit of work is a screening run, not a live feed.

## Architecture

```
ingest (TLEs + launch events)
  -> Postgres (bitemporal)
  -> coordinator splits work
  -> workers propagate and screen
  -> multicast out to API and consumers
```

Ingest runs as CronJobs, Postgres as a StatefulSet with a persistent volume, workers as a Deployment scaled by KEDA. A replay is submitted as a job: an as of timestamp plus screening parameters, which the coordinator fans out across workers.

## Design decisions

**Bitemporal storage.** Every record carries valid time and transaction time. Nothing is overwritten. Prevents lookahead: a query about last Tuesday cannot be answered with data that did not exist last Tuesday. This is what makes replay possible at all.

**Distribution sized for replay, not for one pass.** One screening run is small. A replay sweep across many epochs and thresholds is not, and it is embarrassingly parallel. Phase 2 measures the single machine baseline first so the decision to distribute rests on a number rather than an assumption.

**Pruning filters before propagation.** All pairs is roughly 450M per cycle. Apogee/perigee filter, then orbit path filter, then time filter. Full propagation only runs on what survives.

**UDP multicast with sequence numbers.** Consumers detect gaps and request retransmits from a separate recovery service. Chosen over a broker with guaranteed delivery because loss has to be handled rather than assumed away. Tested with injected loss and delay.

**Autoscaling on queue depth, not CPU.** Replay load is bursty by nature: idle, then a large batch of epochs at once. CPU is a lagging proxy; queue depth is the actual backlog.

**GitOps.** Cluster state lives in Git, reconciled by Argo CD. No manual apply.

## Stack

Python, sgp4, Postgres, Redis or NATS, Docker, Kubernetes (k3s local), KEDA, Prometheus, Grafana, Argo CD, Terraform.

## Roadmap

- [ ] Phase 1: physics core. TLE parsing, SGP4 propagation, coordinate frames, ground tracks, validation against official test vectors
- [ ] Phase 2: screening and the baseline. Close approach detection, pruning filters, full catalog run on one machine, covariance and probability of collision, benchmark harness. Decides whether distribution is warranted.
- [ ] Phase 3: data layer. Postgres schema, bitemporal design, scheduled ingestion, launch event records, as of queries
- [ ] Phase 4: services and replay. API service, containerization, coordinator and workers, replay job submission, multicast publisher and consumers, recovery service, loss and latency testing
- [ ] Phase 5: Kubernetes. Local cluster, StatefulSet and CronJobs, queue depth autoscaling under replay load, Prometheus and Grafana, GitOps, chaos testing
- [ ] Phase 6: depth. Kalman filtering over successive TLEs, frontend with an as of time slider, cloud deployment, writeup with real numbers

Benchmarks go here after Phase 2. Single machine throughput for one pass, replay sweep throughput, p50 and p99, and how they move as workers scale out. If a single machine turns out to handle replay at useful scale, that gets written down here too and the design changes accordingly.

## References

- Vallado, *Fundamentals of Astrodynamics and Applications*
- Vallado et al., *Revisiting Spacetrack Report #3*
- CelesTrak, public catalog data and conjunction assessment writeups
- Kessler and Cour-Palais (1978)
