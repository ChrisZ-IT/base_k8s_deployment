# base_k8s_deployment
base k8s config for networking and LB for on prem/homelab use

## Update calico cidr block
 - run `wget https://raw.githubusercontent.com/projectcalico/calico/refs/heads/master/manifests/calico.yaml` (This will get the latest version of the calico manifest).
   - You can also specify version number like (wget https://raw.githubusercontent.com/projectcalico/calico/refs/tags/v3.29.1/manifests/calico.yaml). [Get list of releases](https://github.com/projectcalico/calico/releases)
 - Update the line `CALICO_IPV4POOL_CIDR` with the desired subnet you want your cluster to run on.(this is the internal subnet your pods will use, This cant be the same subnet as the as your nodes or any other routed subnet on your network)
 - then run `kubectl apply -f calico.yaml`
 - Wait for calico-node and calico-kube-controller pods to enter a running state


## Update metallb config
- Do the [initial metallb setup](https://metallb.io/installation/) by running `kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml`
- update the `addresses` range in `metallb.yml`. This will be the IP address range your LoadBalancer service will use(this allows how you will connect to your pods from outside your cluster) this range can be on the same subnet as you nodes.
- Then run `kubectl apply -f LoadBalancer/metallb.yml` to setup your LoadBalancer base config
- if your metallb speaker pods are having issues starting. Try rebooting each of your k8s nodes one at at time.
