# k3s_metallb_via_k3sup

vagrant-libvirt setup for a k3s cluster without servicelb, so MetalLB can manually be installed 

This is based on what is in the [k3s up and running](https://community.suse.com/courses/4522316/feed) course (see challenge 4 - Replace ServiceLB).

This setup creates three k3s server VMs and one VM as k3s agent (aka worker) and prepares a k3sup installation script via Ansible. This script will install k3s WITHOUT `servicelb`, so you can install MetalLB manually (see below).

Default OS is openSUSE Leap 15.2, but that can be changed in the Vagrantfile. Same holds true for the sizing of the machines.

## Vagrant

1. You need vagrant obviously. And Ansible. And k3sup.
2. Fetch the box, per default this is `opensuse/Leap-15.2.x86_64`, using `vagrant box add opensuse/Leap-15.2.x86_64`.
3. Make sure the git submodules are fully working by issuing `git submodule init && git submodule update`
4. Run `vagrant up`
5. Inspect and run the `k3sup_installation.sh` script
6. Run `kubectl --kubeconfig kubeconfig_k3s_metallb_via_k3sup.yaml get nodes` and you should see your server and agent nodes.
7. Install MetalLB (see below)
8. Party!

## MetalLB installation

The steps to install MetalLB are:
1. create the `metallb-system` namespace
2. deploy MetalLB resources
3. create a shared secret
4. create the configmap to configure MetalLB (especially the IP pool to use)

Depending on the IPs that vagrant-libvirt assigns for your VMs, adjust the following snippet and put it into `metallb-configmap.yaml`. Details can be found in the [metallb documentation](https://metallb.universe.tf/configuration/).
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.240-192.168.1.250
```

Make sure to replace the `v0.9.6` in the following lines with the proper version number of MetalLB.

1. `kubectl --kubeconfig kubeconfig_k3s_metallb_via_k3sup.yaml apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml`
2. `kubectl --kubeconfig kubeconfig_k3s_metallb_via_k3sup.yaml apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml`
3. `kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"`
4. `kubectl --kubeconfig kubeconfig_k3s_metallb_via_k3sup.yaml apply -f metallb-configmap.yaml`

## Cleaning up

When tearing down the the vagrant VMs using `vagrant destroy`, the files created by Ansible are not being removed.

You can do this (using Ansible...) by running `ansible-playbook ansible/playbook-DELETE_everything_to_start_from_scratch.yml`.

This will DELETE all files without asking for confirmation!

## Creating additional agent nodes

You can modify the Vagrantfile to create additional agent nodes by tweaking two lines.

1. Setting the number of agents (in this example to `2`):

```
  ###################################################################################
  # define number of agents
  W = 2
```

2. Adding the additional agent nodes to the `ansible_groups` line:
```
      ansible.groups = {
        "k3sservers"  => [ "k3sserver1", "k3sserver2", "k3sserver3" ],
        "k3sagents"   => [ "k3sagent1", "k3sagent2" ]
      }
```

## Creating additional server nodes

You can modify the Vagrantfile to create additional server nodes by tweaking two lines similar to adjusting the agent number described above.
