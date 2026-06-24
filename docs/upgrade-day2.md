## Upgrade SNO

Trying to upgrade the single node openshift cluster. It failed earlier due to insufficient memory error. This was found in one of the pod events.

### Initial Checks

After starting crc using `crc start` command, check the status of pods on the cluster. Verify if there are any pending pods

Even the logs showed it would take 2 minutes to stabilize, it was taking longer

```sh
INFO Waiting for kube-apiserver availability... [takes around 2min]
INFO Overriding password for developer user
INFO Starting openshift instance... [waiting for the cluster to stabilize]
INFO 3 operators are progressing: console, monitoring, network
INFO 3 operators are progressing: console, monitoring, network
INFO 3 operators are progressing: console, monitoring, network
INFO 5 operators are progressing: console, image-registry, ingress, monitoring, network
INFO 5 operators are progressing: console, image-registry, ingress, monitoring, network
INFO 4 operators are progressing: console, ingress, monitoring, network
INFO 3 operators are progressing: console, monitoring, network
INFO 3 operators are progressing: console, monitoring, network
INFO 3 operators are progressing: console, monitoring, network
INFO 3 operators are progressing: console, monitoring, network
INFO 3 operators are progressing: console, monitoring, network
INFO 3 operators are progressing: console, monitoring, network
INFO 3 operators are progressing: console, monitoring, network
INFO 3 operators are progressing: console, monitoring, network
INFO 3 operators are progressing: console, monitoring, network
WARN Cluster is not ready: cluster operators are still not stable after 10m0.469772257s
INFO Waiting until the user's pull secret is written to the instance disk...
INFO Adding crc-admin and crc-developer contexts to kubeconfig...
Started the OpenShift cluster.
```

From the UI, I could see `Operators` status as `Updating` and `Dynamic Plugins` as `Degraded`.

I used below commands to check plugins

```sh
~$ oc get console.operator.openshift.io cluster -o jsonpath='{.status.conditions[?(@.type=="Degraded")]}'
# No Output

~$ oc get consoleplugins
NAME                        AGE
monitoring-plugin           27h
networking-console-plugin   41d

~$ oc describe consoleplugin monitoring-plugin
Name:         monitoring-plugin
Namespace:
Labels:       app.kubernetes.io/component=monitoring-plugin
              app.kubernetes.io/managed-by=cluster-monitoring-operator
              app.kubernetes.io/name=monitoring-plugin
              app.kubernetes.io/part-of=openshift-monitoring
Annotations:  <none>
API Version:  console.openshift.io/v1
Kind:         ConsolePlugin
Metadata:
  Creation Timestamp:  2026-06-23T07:43:59Z
  Generation:          1
  Resource Version:    60300
  UID:                 43783fb8-8ebe-4ec6-9a31-9e883d900f47
Spec:
  Backend:
    Service:
      Base Path:  /
      Name:       monitoring-plugin
      Namespace:  openshift-monitoring
      Port:       9443
    Type:         Service
  Display Name:   monitoring-plugin
  i18n:
    Load Type:  Preload
Events:         <none>

~$ oc get pods -A | grep -i pending
openshift-console                                  console-6c675887d8-xzfjr                                  0/1     Pending     0              27h
openshift-marketplace                              certified-operators-6spzx                                 0/1     Pending     0              15m
openshift-marketplace                              community-operators-kbkxh                                 0/1     Pending     0              15m
openshift-marketplace                              redhat-marketplace-9bq66                                  0/1     Pending     0              15m
openshift-marketplace                              redhat-operators-hszpr                                    0/1     Pending     0              15m
openshift-monitoring                               metrics-server-78dc549db8-g4c27                           0/1     Pending     0              27h
openshift-monitoring                               monitoring-plugin-7bd56f7ffc-5p4g8                        0/1     Pending     0              27h
openshift-monitoring                               prometheus-k8s-0                                          0/6     Pending     0              27h
openshift-monitoring                               telemeter-client-769654c9b7-88sl8                         0/3     Pending     0              27h
openshift-multus                                   network-metrics-daemon-lqbqs                              0/2     Pending     0              27h
openshift-network-diagnostics                      network-check-source-7f694c8b79-rrg74                     0/1     Pending     0              27h
openshift-network-operator                         iptables-alerter-sl2vc                                    0/1     Pending     0              27h

```

I checked one of the pod events and below is the error in events

```sh
~$ oc describe pod -n openshift-marketplace redhat-marketplace-9bq66

Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  15m                default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint(s). no new claims to deallocate, preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
  Warning  FailedScheduling  13s (x4 over 16m)  default-scheduler  0/1 nodes are available: 1 Insufficient memory. no new claims to deallocate, preemption: 0/1 nodes are available: 1 Insufficient memory.
```

I checked workloads that can be disabled to preserve memory. Below is the list of workloads found using google

- Telemetry and Insights (openshift-monitoring)
- Image Registry (openshift-image-registry)
- Developer Console (openshift-console)
- Adjust Logging and Metrics Retention

#### Scale down Image Registry Pods

```sh

~$ oc get pods -n openshift-image-registry
NAME                                              READY   STATUS    RESTARTS   AGE
cluster-image-registry-operator-8c659bcc5-jvrhd   1/1     Running   4          3d22h
image-registry-79569bcbdb-r2vgm                   1/1     Running   4          3d22h
node-ca-jshrj                                     1/1     Running   4          3d22h

~$ oc scale deployment -n openshift-image-registry cluster-image-registry-operator --replicas=0
Warning: spec.template.spec.nodeSelector[node-role.kubernetes.io/master]: use "node-role.kubernetes.io/control-plane" instead
deployment.apps/cluster-image-registry-operator scaled

~$ oc scale deployment -n openshift-image-registry image-registry --replicas=0
deployment.apps/image-registry scaled

~$ oc get pods -n openshift-image-registry
NAME                                              READY   STATUS    RESTARTS   AGE
cluster-image-registry-operator-8c659bcc5-6lnc6   0/1     Pending   0          31s
node-ca-jshrj                                     1/1     Running   4          3d22h
```

I checked the pods in `Pending` status

```sh
~$ oc get pods -A | grep -i pending
openshift-image-registry                           image-registry-79569bcbdb-4c54f                           0/1     Pending     0              3m24s
openshift-monitoring                               prometheus-k8s-0                                          0/6     Pending     0              27h
openshift-multus                                   network-metrics-daemon-lqbqs                              0/2     Pending     0              27h
openshift-network-operator                         iptables-alerter-sl2vc                                    0/1     Pending     0              27h
```

I tried to disable remote reporting

```sh
~$ oc extract secret/pull-secret -n openshift-config --to=.
.dockerconfigjson

cat .dockerconfigjson | jq 

{
  "auths": {
    "cloud.openshift.com": {
      "auth":

# The above key needs to be removed

~$ jq 'del(.auths."cloud.openshift.com")' .dockerconfigjson > test.json

~$ oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=test.json
secret/pull-secret data updated
""~$ oc get machineconfigpool
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-ab3078c9be3a68c963201967c325163a   True      False      False      1              1                   1                     0                      41d
worker   rendered-worker-b3000cec806c827c2a39746cf8bf430a   True      False      False      0              0                   0                     0                      41d
""~$ oc get machineconfigpool
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-ab3078c9be3a68c963201967c325163a   False     True       False      1              0                   0                     0                      41d
worker   rendered-worker-bb2a651b29963704ff9b4e268006eba2   True      False      False      0              0                   0                     0                      41d
""~$ oc get machineconfigpool
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-c5cf0c292a31a3dc898ecbabeb40936c   True      False      False      1              1                   1                     0                      41d
worker   rendered-worker-bb2a651b29963704ff9b4e268006eba2   True      False      False      0              0                   0                     0                      41d

```

I can see that `Updating` status changed to true for `master` node. After this, telemetry client was no longer running

```sh
~$ oc get pods -n openshift-monitoring
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-main-0                                      6/6     Running   6          28h
cluster-monitoring-operator-768554cd57-r4mvf             1/1     Running   1          28h
kube-state-metrics-7f5d6bf9bb-lm7s7                      3/3     Running   3          28h
metrics-server-78dc549db8-g4c27                          1/1     Running   0          28h
monitoring-plugin-7bd56f7ffc-5p4g8                       1/1     Running   0          28h
node-exporter-thnqp                                      2/2     Running   2          28h
openshift-state-metrics-fbdbd9894-m4wmt                  3/3     Running   3          28h
prometheus-k8s-0                                         0/6     Pending   0          11m
prometheus-operator-7bf89cccc7-8x5sk                     2/2     Running   2          28h
prometheus-operator-admission-webhook-6659cd4f5d-lld6p   1/1     Running   1          28h
thanos-querier-d54f77fbd-mt2js                           6/6     Running   6          28h
```

Tried disabling image registry

```sh
~$ oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Removed"}}'
config.imageregistry.operator.openshift.io/cluster patched

~$ oc get pods -A | grep -i pending
openshift-monitoring                               prometheus-k8s-0                                          0/6     Pending     0              16m
openshift-network-operator                         iptables-alerter-sl2vc                                    0/1     Pending     0              28h
```

Still unable to come out of `Insufficient Memory` issue.

### Reference

- [Disabling Remote Reporting](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/support/remote-health-monitoring-with-connected-clusters#insights-operator-new-pull-secret_remote-health-reporting)