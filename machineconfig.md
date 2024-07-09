

# Components of Machine Config Operator

The Machine Config Operator is a complex component. There are several sub-components and each sub-component performs a different task. This blog does not explain all the sub-components in detail, but you can examine each of them by clicking the links below.

* Machine Config Server
* Machine Config Controller
    * Coordinate upgrade of machines to the desired configuration defined by a MachineConfig Object.
    * Provide options to control upgrades for sets of machines individually.
* Template Controller
    * Generate the MachineConfigs for predefined roles of machines (master, worker).
    * Watch controllerconfig to generate OpenShift-owned MachineConfigs.
* Update Controller
    * Watch if MachineConfigPool .Status.CurrentMachineConfig is updated.
    * Upgrade machines to the desired MachineConfig by coordinating with a daemon running on each machine.
* Render Controller
    * Watch MachineConfigPool object to find all the MachineConfig objects. 
    * Update CurrentMachineConfig with the rendered MachineConfig.
    * Detect changes on all the MachineConfigs and syncs all the MachineConfigPool objects with a new CurrentMachineConfig.
* Kubelet Config Controller
* Machine Config Daemon





# Creating MachineConfig


While creating or modifying machineconfig (mc) object, it is to be ensured that the mode parameter must have an 'octal' value with a leading '0':

```
   filesystem: root
        mode: 0664
        path: /path/to/file
```

These permissions assigned to the file with octal value will get converted into decimal as evident in the output of oc describe mc <mc-name>:

```
    Filesystem:  root
        Mode:  420
        Path:  /path/to/file
```

This is the expected behaviour since the mode accepts value in octal when creating machineconfig file and then it specifies it as decimal value in the machineconfig object.


To troubleshoot this issue use

```
$ oc get mcp -oyaml

$ oc -n openshift-machine-config-operator logs machine-config-daemon-XXXXX 
```




# Log troubleshooting

```
$ oc get mcp worker

NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT
worker   rendered-worker-fe23xxxx                            False     True       True       4              3                   3                     1
```

where we can see one node is degraded (not ready).


```
oc get mcp  worker -o jsonpath='{.status.conditions}'  
```


```
$ export degraded_node=worker-0.example.com
$ oc logs $(oc get pod -o wide |grep $degraded_node |awk '{print $1}') -c machine-config-daemon
```

machine-config-daemon-host.service on a node

```
$ journalctl -f -u machine-config-daemon-host.service
```


Decode URL formated string

Most MachineConfig content source is encoded so in order to see original content, you should decode it.

(EX) crio.conf

```
echo  'urldecode() { : "${*//+/ }"; echo -e "${_//%/\\x}"; }' > ./urldecode.sh; chmod 0775 ./urldecode.sh

./urldecode.sh $(oc get mc rendered-worker-2b30xxxx -o jsonpath='{.spec.config.storage.files[?(@.path=="/etc/crio/crio.conf")].contents.source}' |cut -d',' -f2) > current_mc_crio.conf
```


Check RHCOS OS images

```
rpm-ostree status
```

Change booted image

```
pivot -r $REFSPEC
```




# Order of MachineConfigs

The MCO does things alphanumerically, meaning that the order of application of MachineConfigs does not matter, only the naming of it.

As an example, let’s say I have the following 3 machineconfigs, all writing to the same file:

```
99-worker-chrony-conf-slow
99-worker-chrony-conf-override
100-worker-chrony-conf
```

2 will override 3, since 9>1 and 1 will override 2, since s>o. In the final system, only the file defined in 99-worker-chrony-conf-slow will take effect (again, they are writing to the same file) since the MCO merges them.






Most MachineConfig content source is encoded so in order to see original content, you should decode it.

(EX) crio.conf

```
echo  'urldecode() { : "${*//+/ }"; echo -e "${_//%/\\x}"; }' > ./urldecode.sh; chmod 0775 ./urldecode.sh

./urldecode.sh $(oc get mc rendered-worker-2b30xxxx -o jsonpath='{.spec.config.storage.files[?(@.path=="/etc/crio/crio.conf")].contents.source}' |cut -d',' -f2) > current_mc_crio.conf
```





Check logs of MachineConfigPool to know which nodes have issues and any hints regarding the issues.

Raw
oc get mcp 
oc get mcp  worker -o jsonpath='{.status.conditions}'  
Useful log files
openshift machine config daemon pod log

Raw
$ export degraded_node=worker-0.example.com
$ oc logs $(oc get pod -o wide |grep $degraded_node |awk '{print $1}') -c machine-config-daemon
machine-config-daemon-host.service on a node

Raw
$ journalctl -f -u machine-config-daemon-host.service
Decode URL formated string
Most MachineConfig content source is encoded so in order to see original content, you should decode it.

(EX) crio.conf

Raw
echo  'urldecode() { : "${*//+/ }"; echo -e "${_//%/\\x}"; }' > ./urldecode.sh; chmod 0775 ./urldecode.sh

./urldecode.sh $(oc get mc rendered-worker-2b30xxxx -o jsonpath='{.spec.config.storage.files[?(@.path=="/etc/crio/crio.conf")].contents.source}' |cut -d',' -f2) > current_mc_crio.conf
Check RHCOS OS images
Raw
rpm-ostree status
Change booted image
Raw
pivot -r $REFSPEC







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
