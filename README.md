## Argo CD Scalability Experiments

### Create an Argo CD application from command line
For example:
```shell
argocd app create argocd-scalability-00001 \
--config-management-plugin wrap4kyst-flotta \
--repo https://github.com/IBM-edge-platform-research/kyst-configurations.git \
--path examples/kubernetes/nginx/deploy-flotta \
--dest-server https://$sharedkystmaster:6443  \
--dest-namespace argocd-scalability
```

### Create Argo CD applications using ApplicationSet
Argo CD's [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/) controller can automate the creation of multiple Argo CD applications, using 'generators' and 'templates'.

There are [various generators](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators/) provided by Argo.
Looks like the [Git Generator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/) fits into our use case.
Here is an example:
```shell
kubectl -n argocd apply -f argocd-scalability/applicationset.yaml
```

### Create Argo CD applications using Cluster Loader
[Cluster Loader](https://github.com/kubernetes/perf-tests/tree/master/clusterloader2) can automate the creation of multiple Argo CD applications, because an Argo CD application is a Kubernetes Custom Resource.

However, a little hack is necessary to make it work. The intial trial failed, because Cluster Loader today only allows objects to be created in its managed (meaning created at runtime with randomized names) namespaces. But Argo CD only recogonizes Applications in `argocd` namespace. We can hack Cluster Loader's [code](https://github.com/kubernetes/perf-tests/blob/8a0c339a42a6f0419a10fd3a701a8284c37511f3/clusterloader2/pkg/test/simple_test_executor.go#L181) as a work around:
```go
        nsList = []string{"argocd"}
```
But we still need to manually clean the managed namespaces after running Cluster Loader.
More discussions available [here](https://kubernetes.slack.com/archives/C09QZTRH7/p1657089711061019).

With this work around, we can create Argo CD applications using Cluster Loader, similar to Cluster Loader's getting started [documentation](https://github.com/kubernetes/perf-tests/blob/master/clusterloader2/docs/GETTING_STARTED.md#execute-test):
```shell
go run cmd/clusterloader.go \
  --testconfig <path-to-this-repo>/argocd-scalability/clusterloader-manifests/config.yaml \
  --provider local \
  --kubeconfig <kubeconfig-of-the-cluster-hosting-argo-cd> \
  --v=2 \
  --enable-exec-service=false \
  --delete-automanaged-namespaces=false
```

### Special `wrap4kyst` options for the scalability experiments
We need the `unique-configspec-name` and `configspec-name-suffix` options for the `wrap4kyst` plugin.
Using one of these options, even multiple Argo CD applications are created with one single set of files in git, the output ConfigSpecs are still unique.
Therefore, the multiple Argo CD applications won't conflict with each other.
With these options, we eliminate the need of creating a separate set of files for each Argo CD application.

There is a subtle difference between the `unique-configspec-name` and `configspec-name-suffix` options.
The `configspec-name-suffix` option not only makes the name of the ConfigSpec _object_ unique, but it also makes the name of the _file_ (holding the object) unique.

### Sync an Argo CD application using local directory
```shell
argocd app sync argocd-scalability-00001 --local ./examples/kubernetes/nginx/deploy-flotta/
```

#### Prerequisites
- `kustomize` installed if using `--local` option for `argocd app sync`.
- `wrap4kyst` binary in `PATH` if using the `wrap4kyst` plugin and using the `--local` option for `argocd app sync`. The binary can be made available by compliling the `wrap4kyst` plugin locally, or by copying a complied one from a container:
```shell
docker cp 23058b2b81fb:/wrap4kyst .
```

### Change application resync period
There is a trade-off between the number of Applications and frequency of resync.

The frequency of resync is controlled by resync period with default value of 3 minutes, documented [here](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/#argocd-application-controller).

The resync period can be changed:
```shell
kubectl patch -n argocd configmap argocd-cm --patch-file argocd-scalability/argocd-cm-patch.yaml
```
The application controller needs a restart to pick up the new resync period.

### Change application controller's options
As documented [here](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/#argocd-application-controller), the application controller's 'status-processors' and 'operation-processors' options can be changed:
```shell
kubectl patch -n argocd configmap argocd-cmd-params-cm --patch-file argocd-scalability/argocd-cmd-params-cm-patch.yaml
```
The application controller needs a restart to pick up the new options.
The new options can be seen by running:
```shell
kubectl -n argocd exec argocd-application-controller-0 -- env
```
The partial output is similar to:
```
ARGOCD_RECONCILIATION_TIMEOUT=3m
ARGOCD_APPLICATION_CONTROLLER_STATUS_PROCESSORS=60
ARGOCD_APPLICATION_CONTROLLER_OPERATION_PROCESSORS=30
```

### Main result
In the experiments, we were able to sync 10k Argo CD applications in less than 40 minutes.
Please contact us if interested in more details.
