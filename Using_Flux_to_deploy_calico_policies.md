# GitOps with Calico using Weave Flux

## Setup

Download [fluxctl](https://github.com/fluxcd/flux/releases). 

`export GHUSER=”bikram20”    # Use your GitHub username instead of “bikram20”`

Install flux into your cluster in the namespace "secops". Change secops to a different name of your choice if needed

`kubectl create namespace secops`

Let us install flux into the cluster. Make sure to change the repo path from k8sconfig to your repo name.

```
fluxctl install \
--git-user=${GHUSER} \
--git-email=${GHUSER}@users.noreply.github.com \
--git-url=git@github.com:${GHUSER}/k8sconfig \
--namespace=secops | kubectl apply -f -
```

Flux is a low footprint tool.

```
[centos@ip-172-31-8-215 ~]$ kubectl get all -n secops
NAME                             READY   STATUS    RESTARTS   AGE
pod/flux-5bd78dcf7d-bwm2r        1/1     Running   0          3m31s
pod/memcached-554f994578-lhvws   1/1     Running   0          3m31s


NAME                TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)     AGE
service/memcached   ClusterIP   10.100.9.42   <none>        11211/TCP   3m32s


NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/flux        1/1     1            1           3m32s
deployment.apps/memcached   1/1     1            1           3m32s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/flux-5bd78dcf7d        1         1         1       3m32s
replicaset.apps/memcached-554f994578   1         1         1       3m32s
```

Let us review the logs.

```
[centos@ip-172-31-8-215 ~]$ klon secops pod/flux-5bd78dcf7d-bwm2r --tail=20
ts=2019-10-17T04:02:52.374986889Z caller=warming.go:198 component=warmer info="refreshing image" image=quay.io/tigera/cnx-queryserver tag_count=11 to_update=11 of_which_refresh=0 of_which_missing=11
ts=2019-10-17T04:02:52.599729425Z caller=warming.go:206 component=warmer updated=quay.io/tigera/cnx-queryserver successful=11 attempted=11
ts=2019-10-17T04:02:52.599862597Z caller=images.go:17 component=sync-loop msg="polling for new images for automated workloads"
ts=2019-10-17T04:02:52.599893948Z caller=images.go:27 component=sync-loop msg="no automated workloads"
ts=2019-10-17T04:02:53.019581192Z caller=warming.go:198 component=warmer info="refreshing image" image=quay.io/tigera/es-proxy tag_count=8 to_update=8 of_which_refresh=0 of_which_missing=8
ts=2019-10-17T04:02:53.133970621Z caller=warming.go:206 component=warmer updated=quay.io/tigera/es-proxy successful=8 attempted=8
ts=2019-10-17T04:02:53.134195846Z caller=images.go:17 component=sync-loop msg="polling for new images for automated workloads"
ts=2019-10-17T04:02:53.134216768Z caller=images.go:27 component=sync-loop msg="no automated workloads"
ts=2019-10-17T04:02:53.313915678Z caller=warming.go:198 component=warmer info="refreshing image" image=quay.io/tigera/cnx-manager tag_count=21 to_update=21 of_which_refresh=0 of_which_missing=21
ts=2019-10-17T04:02:53.608476799Z caller=warming.go:206 component=warmer updated=quay.io/tigera/cnx-manager successful=21 attempted=21
ts=2019-10-17T04:02:53.608999201Z caller=images.go:17 component=sync-loop msg="polling for new images for automated workloads"
ts=2019-10-17T04:02:53.609023804Z caller=images.go:27 component=sync-loop msg="no automated workloads"
ts=2019-10-17T04:02:56.300320871Z caller=warming.go:198 component=warmer info="refreshing image" image=quay.io/tigera/cnx-node tag_count=20 to_update=20 of_which_refresh=0 of_which_missing=20
ts=2019-10-17T04:02:56.622637632Z caller=warming.go:206 component=warmer updated=quay.io/tigera/cnx-node successful=20 attempted=20
ts=2019-10-17T04:02:56.622880414Z caller=images.go:17 component=sync-loop msg="polling for new images for automated workloads"
ts=2019-10-17T04:02:56.622919266Z caller=images.go:27 component=sync-loop msg="no automated workloads"
ts=2019-10-17T04:02:56.936254241Z caller=warming.go:198 component=warmer info="refreshing image" image=quay.io/tigera/cnx-apiserver tag_count=12 to_update=12 of_which_refresh=0 of_which_missing=12
ts=2019-10-17T04:02:57.138608285Z caller=warming.go:206 component=warmer updated=quay.io/tigera/cnx-apiserver successful=12 attempted=12
ts=2019-10-17T04:02:57.138801966Z caller=images.go:17 component=sync-loop msg="polling for new images for automated workloads"
ts=2019-10-17T04:02:57.138831269Z caller=images.go:27 component=sync-loop msg="no automated workloads"
ts=2019-10-17T04:05:21.970495279Z caller=loop.go:101 component=sync-loop err="git repo not ready: git clone --mirror: fatal: Could not read from remote repository., full output:\n Cloning into bare repository '/tmp/flux-gitclone835926263'...\ngit@github.com: Permission denied (publickey).\r\nfatal: Could not read from remote repository.\n\nPlease make sure you have the correct access rights\nand the repository exists.\n"
^C
[centos@ip-172-31-8-215 ~]$ 
```

Given that we do not need flux to reach out to registry, nor update any workload... let us disable those. We will basically add 2 options to flux startup in the deployment.

'--registry-exclude-image=' for excluding registry scanning
'--sync-garbage-collection' for deleting the objects deleted from Git.

*-Important note: Review the git config in your flux deployment spec. I had to remove the special characters. Final spec looked as follows.-*

```
    spec:
      containers:
      - args:
        - --memcached-service=
        - --ssh-keygen-dir=/var/fluxd/keygen
        - --git-url=git@github.com:bikram20/k8sconfig
        - --git-branch=master
        - --git-label=flux
        - --git-user=bikram20
        - --git-email=bikram20@users.noreply.github.com
        - --listen-metrics=:3031
        - --registry-exclude-image=*
        - --sync-garbage-collection
```

Now the logs should be much cleaner. Let us take the identity of flux agent and add to GitHub repo's developer settings - ssh keys. Flux will use ssh to connect to the repo.

```
[centos@ip-172-31-8-215 ~]$ klon flux flux-55fd67d5f7-r7dr6     
Error from server (NotFound): namespaces "flux" not found
[centos@ip-172-31-8-215 ~]$ klon secops flux-55fd67d5f7-r7dr6
ts=2019-10-17T04:13:01.299002152Z caller=main.go:248 version=1.15.0
ts=2019-10-17T04:13:01.299083043Z caller=main.go:383 msg="using in cluster config to connect to the cluster"
ts=2019-10-17T04:13:01.31840213Z caller=main.go:468 component=cluster identity=/etc/fluxd/ssh/identity
ts=2019-10-17T04:13:01.318455734Z caller=main.go:469 component=cluster identity.pub="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC9gdHdAdRhF9oiQOGNXtq1ckHgbl2+PveEohD7L8J/gUOPsRGIhjOzJD0eVXijURfwf2XyvQuZAHytu3tKCo2gXBRztKwE/TerKaIgMtl+rjaQJNARqsrtsxcQDixcGA5aA6vryPYJaJ8DXEJY9/uU9zSozYarDXU1aTIKT+zGZ23OniXrGxq4dhuTCkg+UkvFbDLuCJhIZvGBlzZuDHsqo0xgVM95yAfz5vorUdmJ1/P/lc0u4cLtQOAu0cwKC1fMX4XVp1beEAw3cxBm/BnQm30di5RqQUSSeQC4aKK4Owna+x5OexTbH4Lj1sA/Bh6rtseJFsY4s3TleRWG1tAb"
ts=2019-10-17T04:13:01.318484792Z caller=main.go:474 host=https://10.96.0.1:443 version=kubernetes-v1.15.4
ts=2019-10-17T04:13:01.318543928Z caller=main.go:486 kubectl=/usr/local/bin/kubectl
ts=2019-10-17T04:13:01.319595885Z caller=main.go:498 ping=true
ts=2019-10-17T04:13:01.322114526Z caller=main.go:633 url=ssh://git@github.com/%E2%80%9Dbikram20%E2%80%9D/k8sconfig user=”bikram20” email=”bikram20”@users.noreply.github.com signing-key= verify-signatures=false sync-tag=flux state=git readonly=false notes-ref=flux set-author=false git-secret=false
ts=2019-10-17T04:13:01.322175515Z caller=main.go:736 upstream="no upstream URL given"
ts=2019-10-17T04:13:01.322287033Z caller=main.go:765 metrics-addr=:3031
ts=2019-10-17T04:13:01.324861792Z caller=loop.go:101 component=sync-loop err="git repo not ready: git repo has not been cloned yet"
ts=2019-10-17T04:13:01.324905476Z caller=images.go:17 component=sync-loop msg="polling for new images for automated workloads"
ts=2019-10-17T04:13:01.324923816Z caller=images.go:27 component=sync-loop msg="no automated workloads"
ts=2019-10-17T04:13:01.328508533Z caller=main.go:757 addr=:3030
ts=2019-10-17T04:13:01.626105896Z caller=checkpoint.go:24 component=checkpoint msg="up to date" latest=1.15.0


[centos@ip-172-31-8-215 ~]$ fluxctl identity --k8s-fwd-ns secops
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC9gdHdAdRhF9oiQOGNXtq1ckHgbl2+PveEohD7L8J/gUOPsRGIhjOzJD0eVXijURfwf2XyvQuZAHytu3tKCo2gXBRztKwE/TerKaIgMtl+rjaQJNARqsrtsxcQDixcGA5aA6vryPYJaJ8DXEJY9/uU9zSozYarDXU1aTIKT+zGZ23OniXrGxq4dhuTCkg+UkvFbDLuCJhIZvGBlzZuDHsqo0xgVM95yAfz5vorUdmJ1/P/lc0u4cLtQOAu0cwKC1fMX4XVp1beEAw3cxBm/BnQm30di5RqQUSSeQC4aKK4Owna+x5OexTbH4Lj1sA/Bh6rtseJFsY4s3TleRWG1tAb
[centos@ip-172-31-8-215 ~]$

```

Go to your GitHub account, follow the settings for the specific repository, then Deploy Keys, then add the Flux SSH key.

Make sure to enable the checkbox “Allow write access” for the lab use. 


## Verify

First, verify the logs to ensure flux controller connects to Github.

```
[centos@ip-172-31-8-215 ~]$ klon  secops flux-676f84bbbc-rpvfb
ts=2019-10-17T04:24:21.286257041Z caller=main.go:248 version=1.15.0
ts=2019-10-17T04:24:21.286354767Z caller=main.go:383 msg="using in cluster config to connect to the cluster"
ts=2019-10-17T04:24:21.308330809Z caller=main.go:468 component=cluster identity=/etc/fluxd/ssh/identity
ts=2019-10-17T04:24:21.308383432Z caller=main.go:469 component=cluster identity.pub="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC9gdHdAdRhF9oiQOGNXtq1ckHgbl2+PveEohD7L8J/gUOPsRGIhjOzJD0eVXijURfwf2XyvQuZAHytu3tKCo2gXBRztKwE/TerKaIgMtl+rjaQJNARqsrtsxcQDixcGA5aA6vryPYJaJ8DXEJY9/uU9zSozYarDXU1aTIKT+zGZ23OniXrGxq4dhuTCkg+UkvFbDLuCJhIZvGBlzZuDHsqo0xgVM95yAfz5vorUdmJ1/P/lc0u4cLtQOAu0cwKC1fMX4XVp1beEAw3cxBm/BnQm30di5RqQUSSeQC4aKK4Owna+x5OexTbH4Lj1sA/Bh6rtseJFsY4s3TleRWG1tAb"
ts=2019-10-17T04:24:21.308411495Z caller=main.go:474 host=https://10.96.0.1:443 version=kubernetes-v1.15.4
ts=2019-10-17T04:24:21.308456974Z caller=main.go:486 kubectl=/usr/local/bin/kubectl
ts=2019-10-17T04:24:21.309215766Z caller=main.go:498 ping=true
ts=2019-10-17T04:24:21.311257584Z caller=main.go:633 url=ssh://git@github.com/bikram20/k8sconfig user=bikram20 email=bikram20@users.noreply.github.com signing-key= verify-signatures=false sync-tag=flux state=git readonly=false notes-ref=flux set-author=false git-secret=false
ts=2019-10-17T04:24:21.311316467Z caller=main.go:736 upstream="no upstream URL given"
ts=2019-10-17T04:24:21.311438506Z caller=main.go:765 metrics-addr=:3031
ts=2019-10-17T04:24:21.312479312Z caller=images.go:17 component=sync-loop msg="polling for new images for automated workloads"
ts=2019-10-17T04:24:21.312518797Z caller=images.go:27 component=sync-loop msg="no automated workloads"
ts=2019-10-17T04:24:21.312571753Z caller=loop.go:101 component=sync-loop err="git repo not ready: git repo has not been cloned yet"
ts=2019-10-17T04:24:21.313678322Z caller=main.go:757 addr=:3030
ts=2019-10-17T04:24:21.679188843Z caller=checkpoint.go:24 component=checkpoint msg="up to date" latest=1.15.0
ts=2019-10-17T04:24:26.227484128Z caller=loop.go:127 component=sync-loop event=refreshed url=ssh://git@github.com/bikram20/k8sconfig branch=master HEAD=c5672a7f48ca93e83c30ddbade86ca845b8a2aa3
ts=2019-10-17T04:24:32.849776922Z caller=sync.go:479 method=Sync cmd=apply args= count=8
ts=2019-10-17T04:24:33.292685141Z caller=sync.go:545 method=Sync cmd="kubectl apply -f -" took=442.837936ms err=null output="globalnetworkset.projectcalico.org/2-tigera-restricted-resource created\nglobalnetworkset.projectcalico.org/9-public-ip-range created\nnetworkpolicy.networking.k8s.io/access-nginx created\nnetworkpolicy.networking.k8s.io/default-deny-new created\nglobalthreatfeed.projectcalico.org/feodo-tracker created\ntier.projectcalico.org/platform created\ntier.projectcalico.org/security created\nglobalnetworkpolicy.projectcalico.org/security.quarantine created"
ts=2019-10-17T04:24:41.414500228Z caller=daemon.go:685 component=daemon event="Sync: 68fb2e4..c5672a7, <cluster>:globalnetworkpolicy/security.quarantine, <cluster>:globalnetworkset/2-tigera-restricted-resource, <cluster>:globalnetworkset/9-public-ip-range, <cluster>:globalthreatfeed/feodo-tracker, <cluster>:tier/platform, <cluster>:tier/security, policy-demo:networkpolicy/access-nginx, policy-demo:networkpolicy/default-deny-new" logupstream=false
ts=2019-10-17T04:24:43.638589413Z caller=loop.go:220 component=sync-loop state="tag flux" old=054c2976d06b267bd6c8bd4f56438729b94ba11a new=c5672a7f48ca93e83c30ddbade86ca845b8a2aa3
ts=2019-10-17T04:24:43.870124709Z caller=loop.go:127 component=sync-loop event=refreshed url=ssh://git@github.com/bikram20/k8sconfig branch=master HEAD=c5672a7f48ca93e83c30ddbade86ca845b8a2aa3
^C
[centos@ip-172-31-8-215 ~]$
```

Now let's check all the objects (networkpolicy, globalnetworkpolicy, globalnetworkset, globalthreatfeed).

```
[centos@ip-172-31-8-215 ~]$ kubectl get networkpolicy -n policy-demo
NAME               POD-SELECTOR   AGE
access-nginx       run=nginx2     5m38s
default-deny-new   <none>         5m38s
[centos@ip-172-31-8-215 ~]$ 
[centos@ip-172-31-8-215 ~]$ kubectl get globalnetworkpolicy
NAME                  AGE
security.quarantine   5m47s
[centos@ip-172-31-8-215 ~]$ kubectl get globalnetworkset
NAME                           AGE
2-tigera-restricted-resource   5m55s
9-public-ip-range              5m55s
threatfeed.feodo-tracker       5m55s
[centos@ip-172-31-8-215 ~]$ kubectl get globalthreatfeed
NAME            CREATED AT
feodo-tracker   2019-10-17T04:24:33Z
[centos@ip-172-31-8-215 ~]$ 
```

*Further verify by deleting/editing some resources using kubectl. Flux should sync those back to Git.* Also Flux will sync with Git every 5minutes (you really do not need every 5min, every day is good enough for most network policies use cases).


## Tune
- Tune the ClusterRole for flux.
- If you need multiple repo's to sync, then run multiple instances of flux.
- Keep 1 resource per file.
- Order the resources based on dependency (TBD).

