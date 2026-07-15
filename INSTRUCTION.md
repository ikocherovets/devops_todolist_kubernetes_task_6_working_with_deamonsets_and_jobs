# Deployment & Validation Instructions

This document explains how to deploy `daemonset.yml` and `cronjob.yml` to the
cluster, and how to validate that both are working correctly.

## Prerequisites

- `kubectl` configured with access to the target cluster.
- The `todoapp` namespace already exists and contains a running `todoapp`
  Deployment and a `todoapp-service` ClusterIP service (port 80 -> 8080).

## 1. Create the `mateapp` namespace

The DaemonSet and CronJob must live in the `mateapp` namespace:

```bash
kubectl create namespace mateapp
```

(Skip this step if the namespace already exists.)

## 2. Deploy the manifests

```bash
kubectl apply -f .infrastructure/daemonset.yml
kubectl apply -f .infrastructure/cronjob.yml
```

Both resources will be created in the `mateapp` namespace, but they reach
`todoapp-service` in the `todoapp` namespace using the cross-namespace DNS
name `todoapp-service.todoapp.svc.cluster.local`.

## 3. Verify the resources were created

```bash
kubectl get daemonset -n mateapp
kubectl get cronjob -n mateapp
```

You should see `todoapp-daemonset` and `todoapp-cronjob` listed.

## 4. Validate the DaemonSet

Check that a pod is running on each node:

```bash
kubectl get pods -n mateapp -l app=todoapp-daemonset -o wide
```

Tail the logs of one of the pods to confirm it is curling the service every
5 seconds:

```bash
kubectl logs -n mateapp -l app=todoapp-daemonset -f
```

You should see repeated HTTP responses (e.g. HTML/JSON from the ToDo app)
appearing roughly every 5 seconds.

## 5. Validate the CronJob

List the scheduled jobs and confirm the schedule/history settings:

```bash
kubectl get cronjob -n mateapp todoapp-cronjob
```

Expected output includes `*/4 * * * *` as the schedule.

Wait for a job to trigger (up to 4 minutes), then check the job and pod it
created:

```bash
kubectl get jobs -n mateapp
kubectl get pods -n mateapp -l job-name --show-labels
```

View the logs of the most recent job's pod to confirm the health endpoint
was hit successfully:

```bash
kubectl logs -n mateapp <pod-name>
```

The response should be the output of `GET /api/health` from the ToDo app.

To confirm job history retention is working, let the CronJob run for a
while (at least 11 successful executions) and check that only the last 10
successful and 5 failed jobs are retained:

```bash
kubectl get jobs -n mateapp --sort-by=.metadata.creationTimestamp
```

## 6. Trigger a CronJob run manually (optional, for faster testing)

Instead of waiting up to 4 minutes, you can create a one-off Job from the
CronJob's template:

```bash
kubectl create job -n mateapp todoapp-cronjob-manual --from=cronjob/todoapp-cronjob
kubectl logs -n mateapp -l job-name=todoapp-cronjob-manual
```

## 7. Clean up (optional)

```bash
kubectl delete -f .infrastructure/daemonset.yml
kubectl delete -f .infrastructure/cronjob.yml
```