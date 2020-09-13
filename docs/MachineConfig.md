# MachineConfig

MachineConfig objects are very helpful when we need to customize our cluster and change system context
execution.

One of the challenges we found here was to set s sebool property in the all the worker nodes
to allow Freeipa to manage control groups (cgroup) from inside a container.

To learn more about Machine Config Operator (MCO) which has the responsability to apply this
objects, please visit the official repository at:

- https://github.com/openshift/machine-config-operator

The better way to learn about this is by examples, so let's see the MachineConfig that
enable the sebool property that let us to be closer to run the Freeipa container in
a Kuberentes cluster.

[10-enable-container_manage_cgroup-sebool.yaml](../k8s-objects/10-enable-container_manage_cgroup-sebool.yaml).

Let's exaplain the different parts:

- Select the machineconfig pool that is going to apply this machine config. This is
  helpful to apply in a selective way only the nodes which we want to update the
  configutation. In our case we want to apply the configuration only for the
  worker nodes as they are the ones which will run the freeipa container which
  need the systemd execution.

  ```yaml
  metadata:
    labels:
      machineconfiguration.openshift.io/role: worker
  ```

- Define a systemd unit which will execute the necessary commands when the node is
  booted up, to change the control group `container_manage_cgroup` sebool property.
  This is, execute `setsebool container_manage_cgroup true`.

  ```yaml
  systemd:
    units:
      - name: setsebool-container-manage-cgroups.service
        enabled: true
        contents: |
          # Enable container_manage_cgroup so we can run systemd in a container
          # https://developers.redhat.com/blog/2019/04/24/how-to-run-systemd-in-a-container/
          [Unit]
          After=network-online.target
          [Service]
          Type=oneshot
          ExecStart=setsebool container_manage_cgroup true
          RemainAfterExit=yes
          [Install]
          WantedBy=multi-user.target
  ```

- When a MachineConfig is going to be applied to a node or machine pool, is generated a new
  rendered configuration. After generate the new state that is going to be used, the cluster
  automatically start to reboot the nodes in an ordered way. This is, drain the containers
  from a node, reboot the machine applying the new rendered configuration, and when the
  node is ready, enable the node to be schedulable again. This is executed for all the
  nodes affected, and one by one, so that the availability of the cluster is warranted.

- For debug this operator, we have to look at the controller container, this is:

  ```raw
  oc get pods -n openshift-machine-config-operator
  NAME                                         READY   STATUS    RESTARTS   AGE
  etcd-quorum-guard-54cdf5f44d-l878m           1/1     Running   0          2d12h
  etcd-quorum-guard-54cdf5f44d-p65wf           1/1     Running   0          2d12h
  etcd-quorum-guard-54cdf5f44d-rfx9g           1/1     Running   0          2d12h
  machine-config-controller-594ff887c6-b2682   1/1     Running   0          2d12h
  machine-config-daemon-cjdh7                  2/2     Running   1          2d13h
  machine-config-daemon-gc56f                  2/2     Running   1          2d13h
  machine-config-daemon-gfs2x                  2/2     Running   1          2d13h
  machine-config-daemon-htsjm                  2/2     Running   1          2d13h
  machine-config-daemon-lnt6l                  2/2     Running   1          2d13h
  machine-config-daemon-nj5fm                  2/2     Running   1          2d13h
  machine-config-operator-768f8b57db-t8622     1/1     Running   0          2d12h
  machine-config-server-7jtmt                  1/1     Running   0          2d13h
  machine-config-server-fzb98                  1/1     Running   0          2d13h
  machine-config-server-z92v6                  1/1     Running   0          2d13h
  ```

  The one we are into is `machine-config-controller-594ff887c6-b2682`, where we can retrieve information
  about the process applying our MachineConfig. So we only have to watch the logs there, and apply or
  machine config by:

  ```shell
  oc apply -f k8s-objects/10-enable-container_manage_cgroup-sebool.yaml
  oc logs -n openshift-machine-config-operator machine-config-controller-594ff887c6-b2682 -f
  ```

> One situation faced here is that the controller does not print any information at all if something
> is wrong in our MachineConfig object definition at the `config` section. It could be helpful to add
> to the controller some additional information about the cause the object is discarded, to provide
> better feedback and detect that situations properly and proceed with the needed changes to fix it.

## References

- https://docs.openshift.com/container-platform/4.4/nodes/nodes/nodes-nodes-managing.html
- https://github.com/openshift/machine-config-operator
- https://github.com/coreos/fedora-coreos-tracker/issues/368
- https://openshift.tips/machine-config/#disable-auto-rebooting-after-a-change-with-the-machine-config-operator
