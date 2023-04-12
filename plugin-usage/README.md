#### About the `wrap4kyst` Argo CD plugin

`wrap4kyst` is an Argo CD [Config Management Plugin](https://argo-cd.readthedocs.io/en/stable/user-guide/config-management-plugins/).

'kyst' is the name of a project that not publicly visible.
But we don't need to worry about this during the experiments. All we need to do know about kyst are three things:
1. As the name suggests, `wrap4kyst` is just a wrapper that translates Kubernetes native APIs into kyst Custom Resources;
3. Specifically, the two kinds of kyst Custom Resources used here are `ConfigSpec` and `DeviceGroup`;
2. The Custom Resouce Definitions for `ConfigSpec` and `DeviceGroup` are made available [locally in this repository](./crds/).

#### Try the `wrap4kyst` Argo CD plugin
Follow these steps to use the `wrap4kyst` Argo CD plugin.

Install the [kyst CRDs](./crds/) to the destination cluster(s).

Run the following command against the cluster hosting Argo CD to make the `wrap4kyst` binary available in argocd-repo-server pod.
```shell
kubectl patch -n argocd deployment argocd-repo-server --patch-file=plugin-usage/argocd-repo-server-patch.yaml
```

Register the `wrap4kyst` Argo CD plugin.
```shell
kubectl patch -n argocd configmap argocd-cm --patch-file plugin-usage/argocd-cm-patch.yaml
```

Create an Argo CD Application using the plugin. For example:
```shell
argocd app create first-app \
    --config-management-plugin wrap4kyst \
    --repo https://github.com/edge-experiments/gitops-source.git \
    --path kubernetes/guestbook/deploy \
    --dest-server https://172.31.30.2:6443 \
    --dest-namespace default
```

Test the plugin by syncing the Argo CD Application.
```shell
argocd app sync first-app
```

Delete the Argo CD Application.
```shell
argocd app delete first-app
```
