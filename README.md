## Argo CD Scalability Experiments

### Main result
In the experiments, we were able to sync 10k Argo CD Applications in less than 40 minutes.

We have a blog post "[Sync 10,000 Argo CD Applications in One Shot](https://itnext.io/sync-10-000-argo-cd-applications-in-one-shot-bfcda04abe5b)"
to share the procedure of the experiments and the results.

Please contact us if interested in more details.

### Create Argo CD Applications using Cluster Loader 2
[Cluster Loader 2](https://github.com/kubernetes/perf-tests/tree/master/clusterloader2) can automate the creation of multiple Argo CD Applications, because an Argo CD Application is a Kubernetes Custom Resource.

However, a little hack is necessary to make it work. The intial trial failed, because Cluster Loader 2 today only allows objects to be created in its managed (meaning created at runtime with randomized names) namespaces. But Argo CD only recogonizes Applications in `argocd` namespace. We can hack Cluster Loader 2's [code](https://github.com/kubernetes/perf-tests/blob/8a0c339a42a6f0419a10fd3a701a8284c37511f3/clusterloader2/pkg/test/simple_test_executor.go#L181) as a work around:
```go
        nsList = []string{"argocd"}
```
But we still need to manually clean the managed namespaces after running Cluster Loader 2.
More discussions available [here](https://kubernetes.slack.com/archives/C09QZTRH7/p1657089711061019).

With this work around, we can create Argo CD Applications using Cluster Loader 2, similar to Cluster Loader 2's getting started [documentation](https://github.com/kubernetes/perf-tests/blob/master/clusterloader2/docs/GETTING_STARTED.md#execute-test).

Before using this command, make sure the destination-server-address is correctly populated in
clusterloader-manifests/test-application-use-plugin.yaml or clusterloader-manifests/test-application-no-plugin.yaml.
```shell
go run cmd/clusterloader.go \
  --testconfig <path-to-this-repo>/clusterloader-manifests/config.yaml \
  --provider local \
  --kubeconfig <kubeconfig-of-the-cluster-hosting-argo-cd> \
  --v=2 \
  --enable-exec-service=false \
  --delete-automanaged-namespaces=false
```

### Create Argo CD Applications using ApplicationSet
Argo CD's [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/) controller can automate the creation of multiple Argo CD Applications, using 'generators' and 'templates'.

There are [various generators](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators/) provided by Argo.
Looks like the [Git Generator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/) fits into our use case.

As per the Argo CD documentation, 'The Git generator allows you to create Applications based on files within a Git repository, or based on the directory structure of a Git repository.'

Here is an example.
Before using this command, make sure the destination-server-address is correctly populated in applicationset.yaml.
```shell
kubectl -n argocd apply -f applicationset.yaml
```

In the example, two Applications are created from the ApplicationSet.
```shell
argocd app list -l generator=git
```

### Special `wrap4kyst` options for the scalability experiments
We need the `unique-configspec-name` and `configspec-name-suffix` options for the `wrap4kyst` plugin.
Using one of these options, even multiple Argo CD Applications are created with one single set of files in git, the output ConfigSpecs are still unique.
Therefore, the multiple Argo CD Applications won't conflict with each other.
With these options, we eliminate the need of creating a separate set of files for each Argo CD Application.

There is a subtle difference between the `unique-configspec-name` and `configspec-name-suffix` options.
The `configspec-name-suffix` option not only makes the name of the ConfigSpec _object_ unique, but it also makes the name of the _file_ (holding the object) unique.

### Change Application resync period
There is a trade-off between the number of Applications and frequency of resync.

The frequency of resync is controlled by resync period with default value of 3 minutes, documented [here](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/#argocd-application-controller).

The resync period can be changed:
```shell
kubectl patch -n argocd configmap argocd-cm --patch-file argocd-cm-patch.yaml
```
The argocd-application-controller needs a restart to pick up the new resync period.

### Change argocd-application-controller's options
As documented [here](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/#argocd-application-controller), the argocd-application-controller's 'status-processors' and 'operation-processors' options can be changed:
```shell
kubectl patch -n argocd configmap argocd-cmd-params-cm --patch-file argocd-cmd-params-cm-patch.yaml
```
The argocd-application-controller needs a restart to pick up the new options.
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
