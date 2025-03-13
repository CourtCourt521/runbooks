# ExtremelyHighIndividualControlPlaneMemory

## Meaning

This alert becomes pending when a control plane
node is detected to be using more then 90%
of it's total available memory. If the control
plane node continues to use memory above the
90% threshold for 45 minutes then the alert will fire.

## Impact

The memory utilization per instance within control
plane nodes influence the stability,
and responsiveness of the cluster.
This can lead to cluster instability and slow responses from
kube-apiserver or failing requests especially on etcd.
Moreover, OOM kill is expected
which negatively influences the pod scheduling.

## Diagnosis

This alert is triggered by an expression
which consists of a Prometheus query. The
full query is as follows:

```console
(
  1
    -
  sum by (instance) (
  node_memory_MemFree_bytes
  + node_memory_Buffers_bytes
  + node_memory_Cached_bytes
  AND on (instance)
  label_replace( kube_node_role{role="master"}, "instance", "$1", "node", "(.+)" )
  ) / sum by (instance) (
  node_memory_MemTotal_bytes
  AND on (instance)
  label_replace( kube_node_role{role="master"}, "instance", "$1", "node", "(.+)" )
  )
) * 100 > 90
```

The query can be broken down as follows. From the value
of 1 we are subtracting the sum of any node
with the role of master; the free memory left
on the node, the buffer memory on the node,
and the cached memory on the node. This is
divided by the sum of any nodes with the role master.
The total amount of memory available on that node.

We then multiple the resulting value from each
master node by 100 and if any of the
master nodes have a result greater then
90 then the alert will become pending. If
the query continues to return any single master node
above 90 for 45 minutes straight then the alert will fire.

Because this is a comparator operation, if the
condition is not met, there will be no datapoints
displayed by the query.

## Mitigation

- Make sure your control plane nodes
are appropriatly sized for your environment. Red Hat
offers [sizing and scaling guidelines](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/scalability_and_performance/recommended-performance-and-scalability-practices-2)
for clusters of various sizes.

- In the event this alert is firing
and the cluster it extremely unstable, you can
increase the  control plane nodes memory size
to allow the control plane to stabalize so you
can troubleshoot.

- The `oc describe` command will get a large
variety of data on nodes such as current pods
scheduled on the node and it's current
memory/cpu requests. It will also list the
allocated resources total on the node and
how much it's currently using. You can run
a script to get a list of all control plane
nodes and only see the memory and allocation
statistics with an awk script.

  ```console
  $ for i in $(oc get nodes -l node-role.kubernetes.io/control-plane --no-headers | awk '{print $1}'); do    echo " " ;   echo "----------$i description---------"  ;   echo " " ;   oc describe node $i ; done
  ```
- For kubelet-level commands you can get
the memory usage of individual pods by
using the `oc adm top pods` command.
You can further tune it to look at
individual containers by adding the
`--containers=true` flag and also
sort by memory from highest to
lowest by using the `--sort-by=memory` flag.

- For individual troubleshooting nodes
you can ssh or `oc debug` into the control
plane node and use commands such as top
to diagnose memory usage

   ```console
     $ oc debug node/<node>
     $ chroot /host
     $ top -b -n100 -d2
   ```

- You can use Prometheus to deep
dive into the memory usage of nodes and
pods. Red Hat provides multiple pre-built
dashboards and PromQL queries to track
memory usage over time. All within the
`Observe` section of the OpenShift
console.


- Another reason that the memory could
be high on the control planes is because of
`etcd`. Acting at the primary datastore
for the cluster state, etcd is
constantly being  checked and
updated by the Kube API to
match the current cluster state.
An etcd pod runs on each control
plane node. Becaues of the amount of data
and querying etcd keeps track of this can
lead to high memory usage. In order to make
sure etcd is running smoothly. Red Hat offers
several tools for this such as a [script](https://github.com/peterducai/etcd-tools/blob/main/etcd-analyzer.sh)
for analyzing etcd respond times
and amount of objects stored. As well
as the [fio](https://access.redhat.com/solutions/4885641)
which can be used to check the disk performance for the
storage disks backing etcd.

