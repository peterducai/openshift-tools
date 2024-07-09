




# Order of MachineConfigs

The MCO does things alphanumerically, meaning that the order of application of MachineConfigs does not matter, only the naming of it.

As an example, let’s say I have the following 3 machineconfigs, all writing to the same file:

```
99-worker-chrony-conf-slow
99-worker-chrony-conf-override
100-worker-chrony-conf
```

2 will override 3, since 9>1 and 1 will override 2, since s>o. In the final system, only the file defined in 99-worker-chrony-conf-slow will take effect (again, they are writing to the same file) since the MCO merges them.



# How do I find a MCD pod associated with a node

```
oc -n openshift-machine-config-operator get pods --field-selector spec.nodeName=$NODENAME
```




# I used the forcefile (/run/machine-config-daemon-force) to force the upgrade, but nothing is happening

The forcefile does not technically force upgrade, it only skips all validation of configs on the system and then attempts an update regardless of the diff. Depending on your issue at hand, skipping validation may not help you move past your error.



# Logs


Must-gather or “oc adm inspect co/machine-config”
Sosreport for the node that failed to upgrade

Things to look for:

```
systemctl --failed, rpm-ostree status
```

existence of  /etc/pivot/image-pullspec  might imply a failed upgrade
Machine-config-daemon logs:  unexpected on-disk state validating against rendered-master-xxx”
Check if the affected files are modified by customer outside of machine config 
Check if the machineConfig is created correctly 



# Stuck upgrade on a node for no visible reasons

rollback the os image to force MCD to detect the state and upgrade to desired state (rpm-ostree rollback)

Patch the node and set current and desired config to the “the desired config”

```
oc patch node/$NODE -p '{"metadata": {"annotations": {"machineconfiguration.openshift.io/currentConfig": "the desiredConfig"}}}'`
oc patch node/$NODE -p '{"metadata": {"annotations": {"machineconfiguration.openshift.io/desiredConfig": "the desiredConfig"}}}'`
oc patch node $NODE  --type merge --patch '{"metadata": {"annotations": {"machineconfiguration.openshift.io/reason": ""}}}'
pivot -r $REFSPEC
```





# Overriding MCD validation errors

```
touch  /run/machine-config-daemon-force
```

Use it only when you know the issue can be ignored
Force re-evaluation on a node
Delete machine config daemon pod for that node
Check MCD metrics: 
Pivot drain and reboot errors, update status
https://docs.openshift.com/container-platform/4.15/nodes/nodes/nodes-nodes-machine-config-daemon-metrics.html
Node drain issues due to pod disruption budget
Etcd, infra pods -  check the root cause
Customer specific app pods (work with customer)
MCO Troubleshooting Megadoc
https://docs.google.com/document/d/1fgP6Kv1D-75e1Ot0Kg-W2qPyxWDp2_CALltlBLuseec/edit#heading=h.drwt9450c5pc
