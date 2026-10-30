+++
date = '2026-10-29T20:36:04+01:00'
draft = true
title = "Comment lancer un job Kubernetes à partir d'un CronJob ?"
+++

En manipulant des `CronJob` sur Kubernetes, je me suis souvent confronté à la difficulté de tester mes modifications une fois celles-ci appliquées. Si la charge de travail doit être lancée toutes les semaines, j'aimerais bien m'assurer que tout fonctionne au plus tôt.

Une solution naïve consisterait à réduire l'intervalle de temps défini dans le cron, pour le mettre à, disons, 5 minutes. Ce n'est pas idéal, car il faut attendre ce délai avant d'observer l'impact des modifications. De plus, on modifie directement la configuration attendue (pas terrible pour l'approche GitOps), avec le risque d'oublier de revert le changement une fois la validation terminée.

On doit bien pouvoir faire mieux quand même, non ?

Avant toute chose, il faut se souvenir que le concept de `Job` existe dans Kubernetes indépendamment des `CronJob`, qui sont juste un moyen de programmer leur exécution de manière régulière. D'ailleurs, lorsque l'échéance d'un cronjob survient, un job sous-jacent est créé par le contrôleur CronJob de Kubernetes à partir du `jobTemplate`.

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

Petit aparté sur les jobs. In fine, l'objectif est toujours le même : disposer d'une abstraction au-dessus du `Pod` pour en gérer le cycle de vie. Contrairement au `Deployment`, qui vise à garantir la disponibilité d'une charge de travail sur le long terme (parfait pour un serveur web tournant en continu et sans date de fin, par exemple), le `Job` est destiné aux opérations bornées dans le temps.

Revenons à nos moutons : les maintainers de Kubernetes ont pensé à tout, et mettent à notre disposition, par le biais de kubectl, un moyen de définir un job à partir d'un cronjob existant.

```bash
kubectl create job --from=cronjob/<source_cronjob_name> <new_job_name>
```

Avec notre exemple ci-dessus :

```bash
❯ kubectl create job --from=cronjob/my-job my-job-debug
job.batch/my-job-debug created

❯ kubectl get job
NAME              STATUS     COMPLETIONS   DURATION   AGE
my-job-29889411   Complete   1/1           3s         87s
my-job-29889412   Complete   1/1           3s         27s
my-job-debug      Complete   1/1           3s         8s
```

*Note : on voit bien le job créé manuellement à côté des jobs déclenchés via le cron.*

Et le tour est joué, plus besoin de manoeuvres acrobatiques pour déclencher les charges de travail régulières en one-shot.

Point bonus : on peut même aller plus loin et envisager l'exécution à la demande de cronjobs « suspendus ». En effet, la commande `kubectl create job --from ...` fonctionne aussi dans le cas où le cronjob a le paramètre `spec.suspend` valorisé à `true`.

Attention cependant : les modifications portant par exemple sur l'expression du cron, la timezone ou la `concurrencyPolicy` ne sont pas testables directement avec cette méthode. Concernant le premier paramètre, il reste toujours l'excellent [crontab.guru](https://crontab.guru/).
