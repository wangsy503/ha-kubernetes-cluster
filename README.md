Here we created an HA mode K8s cluster with 2 load balancers, 2 controllers and 2 workers. You can change the settings in [`/ha-kubernetes-cluster/host_vars/localhost/defaults.yml`](https://github.com/wangsy503/ha-kubernetes-cluster/blob/master/host_vars/localhost/defaults.yml). The code has been validated under Ubuntu 20.04.6 LTS.

This repository is modified based on [technekey's post](https://technekey.com/automated-kubernetes-cluster-creation-using-libvert-and-kubespray/). Read original post for more information.

## 1 Install requirements in host machine

Install ansible-core(recommend version=2.12.5) and the required ansible collections in your host machine (where you run your ansible playbooks).
```bash
python3 -m pip install --user ansible-core==2.12.5
ansible-galaxy collection install community.libvirt
ansible-galaxy collection install community.crypto
```

## 2 Do the cluster nodes (VMs) creation

Clone my modified cluster generation repository.
```bash
git clone https://github.com/wangsy503/ha-kubernetes-cluster.git
```

Run the cluster creation playbook. You can change the settings inside [`/ha-kubernetes-cluster/host_vars/localhost/defaults.yml`](https://github.com/wangsy503/ha-kubernetes-cluster/blob/master/host_vars/localhost/defaults.yml) before creating the VMs.
```bash
cd ha-kubernetes-cluster/ 
sudo ansible-playbook cluster-provisioner.yml -e cluster_name=development
```
After the VM creation, if you run `virsh list` in your host machine, you should see 6 VMs are running:
```
virsh list

 Id   Name                              State
-------------------------------------------------
 15   development-kube-controller-1     running
 16   development-kube-controller-2     running
 17   development-kube-worker-1         running
 18   development-kube-worker-2         running
 19   development-kube-loadbalancer-1   running
 20   development-kube-loadbalancer-2   running
```


## 3 Use kubespray to create the cluster

Then we can create the K8s cluster. We are going to use [**kubespray**](https://kubespray.io/) (we will clone kubespray repository inside `/development` folder during VM creation), which will call **kubeadm** inside to create the cluster.

More cluster settings can be viewed and modified inside 
- `/ha-kubernetes-cluster/development/kubespray/inventory/development/hosts.yaml`
- `/ha-kubernetes-cluster/development/kubespray/inventory/development/group_vars/k8s_cluster/k8s-cluster.yml`
- `/ha-kubernetes-cluster/development/kubespray/inventory/development/group_vars/all/all.yml`.



```bash
cd development/kubespray 
sudo ansible-playbook -i inventory/development/hosts.yaml --become --become-user=root cluster.yml -u wrds --private-key ../id_ssh_rsa
```

## 4 Validate the changes

Look up the ip address of one of the controller nodes.
```bash
virsh domifaddr development-kube-controller-1
```

Get the ip address, and enter the node using ssh.
```bash
sudo ssh wrds@{controller-ip-addrss} -i ../id_ssh_rs
```

Now you are inside the controller node, check the status of the kubenetes cluster.
```bash
sudo kubectl cluster-info --kubeconfig /etc/kubernetes/admin.conf
```

You should see
<img width="912" alt="Screen Shot 2023-06-18 at 19 36 59" src="https://github.com/wangsy503/ha-kubernetes-cluster/assets/46682066/db7f5461-cfe2-4cbc-9d23-f38b91421f75">


If you run 
```bash
sudo kubectl get node --kubeconfig /etc/kubernetes/admin.conf
```

You should see
<img width="1022" alt="Screen Shot 2023-06-18 at 19 38 29" src="https://github.com/wangsy503/ha-kubernetes-cluster/assets/46682066/d9c59708-02bf-4d9c-8986-307b7cd107ec">

## 5 More playbooks are present to stop, start and delete the cluster.
```bash
#delete the cluster, but save the VM disk

ansible-playbook cluster-delete.yml  -e cluster_to_delete=development

#delete the VM in the cluster and their disks
ansible-playbook cluster-delete.yml  -e cluster_to_delete=development  -e delete_disk=true

#shutdown all the VM in the cluster
ansible-playbook cluster-stop.yml  -e cluster_to_stop=development

#start all the VM in the cluster
ansible-playbook cluster-start.yml  -e cluster_to_start=development
```
