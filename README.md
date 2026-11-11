# Repro: stale same-name TaskRun in Tekton Results → wrong (failed) status in the console PipelineRun topology

Reproducer for the `openshift-pipelines/console-plugin` issue:
**PipelineRun topology shows a task as failed when a stale same-name TaskRun from a previous
PipelineRun is still in Tekton Results** (see Jira SRVKP-XXXX / linked issue).

## Prerequisites
- OpenShift Pipelines installed (tested with operator 1.23.4 / Tekton Pipelines v1.12.3)
- **Tekton Results enabled** (`tekton-results-api` deployment present in `openshift-pipelines`)
- a user with permission to create namespaces + tekton resources
- cluster reachable with `oc`

## Reproduce (3 commands)

```sh
# 0. clone
git clone <this-repo> && cd stale-taskrun-repro

# 1. run #1: task FAILS (exit 1), then delete the PipelineRun
oc apply -k run1-fail
oc wait --for=condition=Succeeded=false pipelinerun/repro-run -n repro-stale-taskrun --timeout=5m
oc delete pipelinerun repro-run -n repro-stale-taskrun --wait=false

# 2. run #2: SAME NAME, task SUCCEEDS (exit 0)
oc apply -k run2-succeed
oc wait --for=condition=Succeeded=True pipelinerun/repro-run -n repro-stale-taskrun --timeout=5m

# 3. open the console:
#    devops->repro-stale-taskrun -> PipelineRuns -> repro-run -> Overview
#    => the "repro-task" node shows a RED "!" (pf-m-danger) and "0/1",
#       although the TaskRun succeeded
```

Verify the API ground truth (both agree the current run is fine):

```sh
oc get taskrun repro-run-repro-task -n repro-stale-taskrun -o jsonpath='{.status.conditions[?(@.type=="Succeeded")].{status,reason}}'
# True Succeeded

# the stale failed record is still in Tekton Results (filter by the old run UID):
oc get pipelinerun repro-run -n repro-stale-taskrun -o jsonpath='{.metadata.uid}'
```

## Why it happens (short)
The console plugin fetches the run's TaskRuns from the K8s API **and** from the Tekton
Results API. The Results query filters only by label
`tekton.dev/pipelineRun=<name>` (no UID), so the record left by the **deleted** run #1
(failed) comes back together with the record of the current run #2 (succeeded). The merge
dedupes by `metadata.uid` (both survive) and the task status is taken from the **first**
match by `tekton.dev/pipelineTask` label — which is the stale failed one.

Relevant code: `pipelineRuns-details/PipelineRunVisualization.tsx` (does not pass
`pipelineRunUid`), `hooks/useTaskRuns.ts` (`uniqBy(..., metadata.uid)`),
`utils/pipeline-utils.ts` (`appendPipelineRunStatus` → `_.find` by task label).

## Cleanup

```sh
oc delete namespace repro-stale-taskrun
```

(Optionally also purge the stale records from Tekton Results if you want the cluster fully
clean; the bug will simply not be reproducible in that namespace afterwards.)

## Layout

| path | what |
|---|---|
| `base/` | namespace `repro-stale-taskrun` + minimal `Pipeline` (one task, exit code via param) |
| `run1-fail/` | PipelineRun `repro-run` with `exit-code=1` (the run to delete) |
| `run2-succeed/` | PipelineRun `repro-run` (same name) with `exit-code=0` (the run to inspect) |
| `kustomization.yaml` | root — just `base`, for `oc apply -k .` (namespace + pipeline only) |
