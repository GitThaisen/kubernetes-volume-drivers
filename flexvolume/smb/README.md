# CIFS/SMB FlexVolume driver for Kubernetes (Preview)
 - supported Kubernetes version: available from v1.7
 - supported agent OS: Linux 

# About
This driver allows Kubernetes to access SMB server by using [CIFS/SMB](https://en.wikipedia.org/wiki/Server_Message_Block) protocol.

# Install smb FlexVolume driver on a kubernetes cluster
## 1. config kubelet service to enable FlexVolume driver
> Note: skip this step in [AKS](https://azure.microsoft.com/en-us/services/container-service/) or from [acs-engine](https://github.com/Azure/acs-engine) v0.12.0

Please refer to [config kubelet service to enable FlexVolume driver](https://github.com/Azure/kubernetes-volume-drivers/blob/master/flexvolume/README.md#config-kubelet-service-to-enable-flexvolume-driver)
 
## 2. install smb FlexVolume driver on every agent node
### Option#1. Automatically install by k8s daemonset
create daemonset to install smb driver
```
kubectl create -f https://raw.githubusercontent.com/knan-nrk/kubernetes-volume-drivers/d1ab0815f24ced8ef80009cb169247fe5fceca35/flexvolume/smb/deployment/smb-flexvol-installer.yaml
```
> Note: You may replace `/etc/kubernetes/volumeplugins` with `/usr/libexec/kubernetes/kubelet-plugins/volume/exec/`(by default) in `smb-flexvol-installer.yaml` if it's not a k8s cluster on Azure.

 - check daemonset status:
```
watch kubectl describe daemonset smb-flexvol-installer --namespace=flex
watch kubectl get po --namespace=flex -o wide
```
> Note: for deployment on v1.7, it requires restarting kubelet on every node(`sudo systemctl restart kubelet`) after daemonset running complete due to [Dynamic Plugin Discovery](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md#dynamic-plugin-discovery) not supported on k8s v1.7

### Option#2. Manually install on every agent node
```
sudo mkdir -p /etc/kubernetes/volumeplugins/microsoft.com~smb/

cd /etc/kubernetes/volumeplugins/microsoft.com~smb
sudo wget -O smb https://raw.githubusercontent.com/Azure/kubernetes-volume-drivers/master/flexvolume/smb/deployment/smb-flexvol-installer/smb
sudo chmod a+x smb
```

## 3. install `jq` package on every agent node
```
sudo apt install jq -y
```

# Basic Usage
## 1. create a secret which stores smb account name and password
```
kubectl create secret generic smbcreds --from-literal username=USERNAME --from-literal password="PASSWORD" --type="microsoft.com/smb"
```

## 2. create a pod with smb flexvolume mount on linux
#### Option#1 Ties a flexvolume volume explicitly to a pod
- download `nginx-flex-smb.yaml` file and modify `source` field
```
wget -O nginx-flex-smb.yaml https://raw.githubusercontent.com/Azure/kubernetes-volume-drivers/master/flexvolume/smb/nginx-flex-smb.yaml
vi nginx-flex-smb.yaml
```
 - create a pod with smb flexvolume driver mount
```
kubectl create -f nginx-flex-smb.yaml
```

#### Option#2 Create smb flexvolume PV & PVC and then create a pod based on PVC
 > Note: access modes of smb PV supports ReadWriteOnce(RWO), ReadOnlyMany(ROX) and ReadWriteMany(RWX)
 - download `pv-smb-flexvol.yaml` file, modify `source` field and create a smb flexvolume persistent volume(PV)
```
wget https://raw.githubusercontent.com/Azure/kubernetes-volume-drivers/master/flexvolume/smb/pv-smb-flexvol.yaml
vi pv-smb-flexvol.yaml
kubectl create -f pv-smb-flexvol.yaml
```

 - create a smb flexvolume persistent volume claim(PVC)
```
 kubectl create -f https://raw.githubusercontent.com/Azure/kubernetes-volume-drivers/master/flexvolume/smb/pvc-smb-flexvol.yaml
```

 - check status of PV & PVC until its Status changed from `Pending` to `Bound`
 ```
 kubectl get pv
 kubectl get pvc
 ```
 
 - create a pod with smb flexvolume PVC
```
 kubectl create -f https://raw.githubusercontent.com/Azure/kubernetes-volume-drivers/master/flexvolume/smb/nginx-flex-smb-pvc.yaml
 ```

## 3. enter the pod container to do validation
 - watch the status of pod until its Status changed from `Pending` to `Running`
```
watch kubectl describe po nginx-flex-smb
```
 - enter the pod container
```
kubectl exec -it nginx-flex-smb -- bash
```

```
root@nginx-flex-smb:/# df -h
Filesystem                                 Size  Used Avail Use% Mounted on
overlay                                    291G  3.2G  288G   2% /
tmpfs                                      3.4G     0  3.4G   0% /dev
tmpfs                                      3.4G     0  3.4G   0% /sys/fs/cgroup
//xiazhang3.file.core.windows.net/k8stest   25G   64K   25G   1% /data
/dev/sda1                                  291G  3.2G  288G   2% /etc/hosts
shm                                         64M     0   64M   0% /dev/shm
tmpfs                                      3.4G   12K  3.4G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                                      3.4G     0  3.4G   0% /sys/firmware
```
In the above example, there is a `/data` directory mounted as smb filesystem.

#### Debugging skills
 - If there is pod mounting error like following:
```
MountVolume.SetUp failed for volume "test" : invalid character 'C' looking for beginning of value
```
Please attach log file `/var/log/smb-driver.log` and file an issue

### Links
[CIFS/SMB wiki](https://en.wikipedia.org/wiki/Server_Message_Block)

[Flexvolume doc](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md)

[Persistent Storage Using FlexVolume Plug-ins](https://docs.openshift.org/latest/install_config/persistent_storage/persistent_storage_flex_volume.html)
