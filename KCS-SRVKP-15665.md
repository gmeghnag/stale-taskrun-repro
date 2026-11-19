# Title
Red Hat OpenShift Pipelines console shows a PipelineRun task as failed when a same-name PipelineRun was previously deleted in RHOCP 4

# Issue
- In the Red Hat OpenShift Pipelines console, the PipelineRun details page (topology) shows a task with a red "!" icon and a step count of `0/1`, indicating that the task failed.
- The TaskRun for that task actually succeeded: `oc get taskrun` reports `Succeeded=True`.
- The PipelineRun itself completed with status `Succeeded`.
- The problem appears after a previous PipelineRun with the same name was deleted from the cluster.
- Only the task whose name matches the previous run is displayed incorrectly; the other task nodes in the topology are correct.

# Environment
- Red Hat OpenShift Container Platform (RHOCP)
  - 4.21
- Red Hat OpenShift Pipelines
  - 1.24 (verified on 1.24.3; the same code path is present in 1.22, 1.23 and 1.25 by code inspection, not verified at runtime)
- Tekton Results enabled (deployment `tekton-results-api` in the namespace `openshift-pipelines`)

# Resolution
The pipeline execution itself is not affected; only the console display is wrong. Until a fix is released, apply one of the following workarounds:
1. workaround: do not reuse the same PipelineRun name. Create each PipelineRun with a unique name, for example by appending a timestamp.
2. workaround: if the same name must be reused, purge the TaskRun records of the previous PipelineRun for that namespace from the Tekton Results archive before creating the new PipelineRun, so that only the current TaskRun remains.

A fix is tracked in SRVKP-15665: https://redhat.atlassian.net/browse/SRVKP-15665

# Root Cause
The OpenShift Pipelines console builds the PipelineRun topology by merging TaskRun data from the cluster API and from the Tekton Results archive. The Results query of the topology view filters by the PipelineRun name only and not by the PipelineRun UID, so the TaskRun record left in Results by a previously deleted PipelineRun with the same name matches the query as well. When several TaskRun records share the same task name, the console takes the state from a single record and displays the state of the stale record instead of the state of the current TaskRun.

# Diagnostic Steps
1. Verify the actual state of the TaskRun:
   ```
   oc get taskrun <taskrun-name> -n <namespace> \
     -o jsonpath='{.status.conditions[0].status} {.status.conditions[0].reason}'
   ```
   Output:
   ```
   True Succeeded
   ```
2. Verify the PipelineRun state:
   ```
   oc get pipelinerun <pipelinerun-name> -n <namespace> \
     -o jsonpath='{.status.conditions[0].status} {.status.conditions[0].reason}'
   ```
   Output:
   ```
   True Succeeded
   ```
3. Confirm that the console topology node for the task shows the red "!" icon and the `0/1` badge while the API reports success.
4. Confirm that Tekton Results is enabled and that a previous PipelineRun with the same name was deleted, which leaves a stale TaskRun record in the Results archive.
5. List the TaskRun records that Tekton Results returns for the PipelineRun name (label filter only, no UID):
   ```
   oc get --raw "/api/v1/namespaces/openshift-pipelines/services/tekton-results-api-service:8080/proxy/apis/results.tekton.dev/v1alpha2/parents/<namespace>/results/-/records?page_size=50&filter=data_type==%22tekton.dev/v1.TaskRun%22%20%26%26%20data.metadata.labels%5B%22tekton.dev/pipelineRun%22%5D==%22<pipelinerun-name>%22"
   ```
   Several records with the same `metadata.name` but different `metadata.uid` and different `status.conditions[0].reason` values confirm that stale records are present.
6. Add the `tekton.dev/pipelineRunUID` condition with the UID of the current PipelineRun and confirm that only one record is returned. This isolates the current TaskRun and demonstrates the missing filter:
   ```
   oc get pipelinerun <pipelinerun-name> -n <namespace> -o jsonpath='{.metadata.uid}'
   oc get --raw "/api/v1/namespaces/openshift-pipelines/services/tekton-results-api-service:8080/proxy/apis/results.tekton.dev/v1alpha2/parents/<namespace>/results/-/records?page_size=50&filter=data_type==%22tekton.dev/v1.TaskRun%22%20%26%26%20data.metadata.labels%5B%22tekton.dev/pipelineRun%22%5D==%22<pipelinerun-name>%22%20%26%26%20data.metadata.labels%5B%22tekton.dev/pipelineRunUID%22%5D==%22<pipelinerun-uid>%22"
   ```

# Tags
shift_pipelines, ocp_4

# Internal Tags
SRVKP-15665, openshift-pipelines, tekton-results, console-plugin, ocp_4
