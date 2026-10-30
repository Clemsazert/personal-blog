+++
date = '2026-10-29T20:36:04+01:00'
draft = true
title = 'How to Run a Kubernetes Job from a CronJob'
+++

When working with `CronJob` resources on Kubernetes, I have often struggled to test my changes once they were applied. If the workload is meant to run once a week, I would rather make sure everything works as soon as possible.

A naive solution would be to shorten the interval defined in the cron schedule, setting it to, say, 5 minutes. This is not ideal, since you still have to wait for that delay before seeing the effect of your changes. On top of that, you end up modifying the desired configuration directly (not great with a GitOps approach), with the risk of forgetting to revert the change once validation is done.

Surely we can do better, right?

First of all, remember that the `Job` concept exists in Kubernetes independently of `CronJob` resources, which are simply a way to schedule their execution on a regular basis. In fact, whenever a cronjob's schedule comes due, the Kubernetes CronJob controller creates an underlying job from the `jobTemplate`.

```bash
❯ kubectl create cronjob my-job --image=busybox --schedule="*/1 * * * *" -- date
cronjob.batch/my-job created

❯ kubectl get cronjob
NAME     SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
my-job   */1 * * * *   <none>     False     0        <none>          10s

❯ kubectl get job
NAME              STATUS    COMPLETIONS   DURATION   AGE
my-job-29889402   Running   0/1           4s         4s
```

A quick aside on jobs. Ultimately, the goal is always the same: having an abstraction on top of the `Pod` to manage its lifecycle. Unlike a `Deployment`, which aims to keep a workload available over the long run (perfect for a web server running continuously with no end date, for example), a `Job` is meant for time-bounded operations.

Back to the matter at hand: the Kubernetes maintainers have thought of everything, and through kubectl they give us a way to define a job from an existing cronjob.

```bash
kubectl create job --from=cronjob/<source_cronjob_name> <new_job_name>
```

With our example above:

```bash
❯ kubectl create job --from=cronjob/my-job my-job-debug
job.batch/my-job-debug created

❯ kubectl get job
NAME              STATUS     COMPLETIONS   DURATION   AGE
my-job-29889411   Complete   1/1           3s         87s
my-job-29889412   Complete   1/1           3s         27s
my-job-debug      Complete   1/1           3s         8s
```

*Note: the manually created job shows up right next to the jobs triggered by the cron schedule.*

And that's it: no more acrobatics needed to trigger scheduled workloads as one-offs.

Bonus: you can go even further and run "suspended" cronjobs on demand. The `kubectl create job --from ...` command also works when the cronjob has its `spec.suspend` field set to `true`.

A word of caution, though: changes to things like the cron expression, the timezone, or the `concurrencyPolicy` cannot be tested directly with this method. For the first one, there is always the excellent [crontab.guru](https://crontab.guru/).
