# Kyverno Demo
Using Kyverno as an Admission Controller. Review official [docs](https://kyverno.io/docs/) for latest information. We will use Kyverno to deploy validating and mutating webhooks for k8s API requests of select resources. In this example we use validating webhooks to ensure we have a specific label on our Pod resources and a mutating web hook to add a label on resources

## Installing Kyverno
Although installation via Helm is an option we will install using a manifest. Review installation [docs](https://kyverno.io/docs/installation/) for compatibility and versioning.

```bash
kubectl create -f https://raw.githubusercontent.com/kyverno/kyverno/main/config/release/install.yaml
```
<br>

This will deploy the latest main branch so edit the target file if a specific release is required

## Validating Webhook
We will create a new NameSpace `dev` for testing purposes. Then we only allow creation of Pods if it has a label with key 'owner' by creating a policy using the `Policies.kyverno.io` custom resource:

```bash
# Create the dev NameSpace
kubectl create ns dev

# Create the valdiating webhook policy
kubectl apply ./yaml/check-labels-policy.yaml
```

You can check the logs on the kyverno controller pod and look for any errors. Here we will open a new session and tail the log:

```bash
kubectl -n kyverno logs <kyverno_pod_name> -f
```

Now we will try to deploy a pod without the required `owner` label

```bash
kubectl apply -f ./yaml/pod-nolabel.yaml
```
>you should receive output similar to:
```
Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:

resource Pod/dev/netshoot was blocked due to the following policies

require-labels:
  check-for-labels: 'validation error: The label `owner` is required. Rule check-for-labels
    failed at path /metadata/labels/owner/'
```
Now we will deploy the pod again but with the expected label:
```bash
kubectl apply -f ./yaml/pod-withlabel.yaml
```
>the pod should now be created and we can check the labels

``` bash
kubectl get pods -n dev --show-labels

NAME                         READY   STATUS    RESTARTS   AGE    LABELS
netshoot                     1/1     Running   0          116s   app=netshoot,owner=k8s-lover
```

## Mutating Webook
We will create a cluster policy to add a label of the name of the kubectl user to any pods created in the `default` NameSpace.

```bash
#create kyvernon clusterpolicy 
kubectl apply -f ./yaml/add-label-clusterpolicy.yaml
```
We now deploy another pod into the `default` namespace
```bash
kubectl apply -f ./yaml/pod-default-ns.yaml
```
now we can check what labels were applied
```bash
kubectl get pods netshoot --show-labels
```
>we should now see a label `created by` with the name of the user initiating the pod creation:
```bash
NAME       READY   STATUS    RESTARTS   AGE   LABELS
netshoot   1/1     Running   0          6s    app=netshoot,created-by=masterclient
```

## Uninstallation
Review uninstall [docs](https://kyverno.io/docs/installation/#option-2---uninstall-kyverno-with-helm) if required.

```bash
#delete the pods and dev namespace
kubectl delete -f ./yaml/pod-withlabel.yaml
kubectl delete -f ./yaml/pod-default-ns.yaml
kubectl delete ns dev

#uninstall kyverno (use the same installation manifest to delete)
kubectl delete -f https://raw.githubusercontent.com/kyverno/kyverno/main/definitions/release/install.yaml

#clean webhook configurations
kubectl delete mutatingwebhookconfigurations kyverno-policy-mutating-webhook-cfg kyverno-resource-mutating-webhook-cfg kyverno-verify-mutating-webhook-cfg

kubectl delete validatingwebhookconfigurations kyverno-policy-validating-webhook-cfg kyverno-resource-validating-webhook-cfg
```