## Day3

I deleted existing crc setup using `crc delete` command due to `Insufficient Memory` error. I started again to see how I can run openshift SNO with less resources.

After reading further articles, I confirmed that we cannot upgrade SNO setup using crc. Instead, we can delete single node and then setup new version.

I set memory for crc using below command

```sh
crc config set memory 20480
```

The pods were fine. No pending pods. 

I checked if logging is enabled. 

```sh
~$ oc get csv -n openshift-logging
No resources found in openshift-logging namespace.
```

I found image-registry is running. Disable it to free up resources

```sh
~$ oc get pods -n openshift-image-registry
NAME                                              READY   STATUS    RESTARTS   AGE
cluster-image-registry-operator-8c659bcc5-zjrnt   1/1     Running   2          75m
image-registry-79569bcbdb-w7kw6                   1/1     Running   2          74m
node-ca-tf9q2                                     1/1     Running   2          75m

~$ oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Removed"}}'
config.imageregistry.operator.openshift.io/cluster patched

~$ oc scale deployment -n openshift-image-registry cluster-image-registry-operator --replicas=0
Warning: spec.template.spec.nodeSelector[node-role.kubernetes.io/master]: use "node-role.kubernetes.io/control-plane" instead
deployment.apps/cluster-image-registry-operator scaled

## After few minutes

~$ oc get pods -n openshift-image-registry
NAME                                              READY   STATUS    RESTARTS   AGE
cluster-image-registry-operator-8c659bcc5-zjrnt   1/1     Running   2          80m
node-ca-tf9q2                                     1/1     Running   2          80m
```

Checked the resource consumption and Ubuntu VM seems fine.
