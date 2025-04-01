# Portworx-CSI Demo

## General overview

This demo setup is using [px-deploy](https://github.com/PureStorage-OpenConnect/px-deploy) to deploy a Kubernetes cluster on VMware vSphere. The Kubernetes cluster has one master node and three worker nodes. Within the deployment there are pretty cool custom aliases set for the Kubernetes context. 
The underlying storage will be a Pure Storage FlashArray, connected via iSCSI. 


## Using  px-deploy to deploy a K8 cluster

### Create K8 VMs on vSphere

```bash
px-deploy create -n px-csi-demo --cloud vsphere
```

### Check status of deployment
```bash
px-deploy status -n px-csi-demo
```

## Add network adapters to worker nodes in vSphere

By now we have to do it manually. Maybe we'll customize the px-deploy templates and Terraform to adapt a dual-nic template deployment in the future.

Make sure the added NIC will have a network connectivity via iSCSI/NFS to your desired FlashArray or FlashBlade. 

```bash
ping 192.168.200.101
```


## Configure multipathing

The required multipathing binaries should already installed with the latest px-deploy version. If not, have a look at the [miscellaneous](#miscellaneous) chapter at the bottom.

https://docs.portworx.com/portworx-csi/install/prepare/flash-array

Create the `multipath.conf` file:
```bash
nano /etc/multipath.conf
```

Paste the following lines into it:
```bash
defaults {
    user_friendly_names no
    enable_foreign "^$"
        polling_interval    10
}

devices {
    device {
        vendor                      "NVME"
        product                     "Pure Storage FlashArray"
        path_selector               "queue-length 0"
        path_grouping_policy        group_by_prio
        prio                        ana
        failback                    immediate
        fast_io_fail_tmo            10
        user_friendly_names         no
        no_path_retry               0
        features                    0
        dev_loss_tmo                60
        find_multipaths             yes
    }
    device {
        vendor                   "PURE"
        product                  "FlashArray"
        path_selector            "service-time 0"
        hardware_handler         "1 alua"
        path_grouping_policy     group_by_prio
        prio                     alua
        failback                 immediate
        path_checker             tur
        fast_io_fail_tmo         10
        user_friendly_names      no
        no_path_retry            0
        features                 0
        dev_loss_tmo             600
        find_multipaths          yes
    }
}

blacklist_exceptions {
        property "(SCSI_IDENT_|ID_WWN)"
}

blacklist {
      devnode "^pxd[0-9]*"
      devnode "^pxd*"
      device {
        vendor "VMware"
        product "Virtual disk"
      }
}
```

Restart services to apply the new config file:

```bash
systemctl restart multipathd.service
```



Use the following oneliner to automate:

```bash
for node in node-1-1 node-1-2 node-1-3; do
    ssh "$node" "cat <<EOF | sudo tee /etc/multipath.conf
defaults {
    user_friendly_names no
    enable_foreign \"^$\"
        polling_interval    10
}

devices {
    device {
        vendor                      \"NVME\"
        product                     \"Pure Storage FlashArray\"
        path_selector               \"queue-length 0\"
        path_grouping_policy        group_by_prio
        prio                        ana
        failback                    immediate
        fast_io_fail_tmo            10
        user_friendly_names         no
        no_path_retry               0
        features                    0
        dev_loss_tmo                60
        find_multipaths             yes
    }
    device {
        vendor                   \"PURE\"
        product                  \"FlashArray\"
        path_selector            \"service-time 0\"
        hardware_handler         \"1 alua\"
        path_grouping_policy     group_by_prio
        prio                     alua
        failback                 immediate
        path_checker             tur
        fast_io_fail_tmo         10
        user_friendly_names      no
        no_path_retry            0
        features                 0
        dev_loss_tmo             600
        find_multipaths          yes
    }
}

blacklist_exceptions {
        property \"(SCSI_IDENT_|ID_WWN)\"
}

blacklist {
      devnode \"^pxd[0-9]*\"
      devnode \"^pxd*\"
      device {
        vendor \"VMware\"
        product \"Virtual disk\"
      }
}
EOF
sudo systemctl restart multipathd"
done
```

## Connect to FlashArray

At this point you need to create an API key/token to make the iSCSI connections work. You can do this the manual way via GUI/CLI or use my automated script [FlashArray-Create-API-Key](https://github.com/stschappo/FlashArray-Create-API-Key)

### Create pure.json (from master node)

Create the pure.json file on the master node:

```bash
nano pure.json
```

Paste the following into it (update it with your specific values).

```json
{
  "FlashArrays": [
    {
      "MgmtEndPoint": "your-flasharray-mgmt-ip",
      "APIToken": "your-api-token-here"
    }
  ]
}
```

## Create K8 namespace & secret

### Create namespace

```bash
kubectl create ns portworx
```

### Create secret with the API token

```bash
kubectl create secret generic px-pure-secret --namespace portworx --from-file=pure.json=/root/pure.json
```

### Remove pure.json

As the json file is not needed anymore you can delete it:

```bash
rm /root/pure.json
```


## Portworx Central Create CSI Spec

Login or Register to [Portworx Central](https://central.portworx.com/) and create a new PX-CSI spec. 

Enter your correct K8 version, namespace, SAN, etc.

In this case, we used the following settings:

| **Parameter**                       | **Value**  |
| ----------------------------------- | ---------- |
| PX-CSI Version                      | 25.2.0     |
| Distribution Name                   | None       |
| K8s Version                         | 1.31.6     |
| Namespace                           | portworx   |
| Cluster Name Prefix                 | px-cluster |
| Storage Area Network                | iSCSI      |
| Starting Port for Portworx Services | 9001       |
| Telemetry                           | Disabled   |


### Install the Portworx Operator

```bash
kubectl apply -f 'https://install.portworx.com/25.2.0?comp=pxoperator&oem=px-csi&kbver=1.31.6&ns=portworx'
```

### Download  StorageCluster YAML

Replace with your own URL:

```bash
curl -o stc.yaml 'https://install.portworx.com/25.2.0?oem=px-csi&operator=true&ce=pure&csi=true&stork=false&mon=true&promop=true&kbver=1.31.6&ns=portworx&c=px-cluster-a141f05e-18f0-4ade-bb93-adacdbc44461&r=9001&pureSanType=ISCSI&tel=false'
```

### Skip health-check in StorageCluster spec until a fix is out in px-deploy

Oneliner to set the desired annotations:

```bash
sed -i 's/portworx.io\/misc-args: "--oem px-csi"/portworx.io\/misc-args: "--oem px-csi"\n    portworx.io\/health-check: skip/' stc.yaml
```

Apply the new config:

```bash
kubectl apply -f stc.yaml
```

## Miscellaneous

Check Storage Cluster:

```bash
kubectl get storagecluster -n portworx
```

Check Storage Cluster (verbose):

```bash
kubectl describe storagecluster -n portworx
```

Get iqn from the worker nodes:

```bash
cat /etc/iscsi/initiatorname.iscsi | grep iqn
```

Get iqn from all worker nodes parallel:

```bash
for node in node-1-{1..3}; do ssh "$node" "cat /etc/iscsi/initiatorname.iscsi | grep iqn"; done
```

Install the required multipathing binaries on a worker node locally (reboot required):

```bash
dnf install -y sg3_utils device-mapper-multipath iscsi-initiator-utils
```

Install the required multipathing binaries on all worker nodes and reboot them:

```bash
for node in node-1-1 node-1-2 node-1-3; do ssh "$node" "dnf install -y sg3_utils device-mapper-multipath iscsi-initiator-utils && reboot" & done
```


## Troubleshooting

A list of common issues:

- Mismatching Kubernetes version in Portworx spec
- No network adapter for iSCSI connection
- No connection to iSCSI target
- Skip-health-check annotation not set
- Wrong IP in the pure.json file
