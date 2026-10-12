# base_k8s_deployment
base k8s config for networking and LB for on prem/homelab use

## Installing my CNI
 I went with calico for my K8s CNI and I am using the [helm deployment](https://docs.tigera.io/calico/latest/getting-started/kubernetes/helm) for this. Use official docs over this one.

  - Add the helm repo: `helm repo add projectcalico https://docs.tigera.io/calico/charts && helm repo update`
  - Create the namespace: `k create namespace tigera-operator`
  - Install the calico CRDs: `helm template calico-crds projectcalico/projectcalico.org.v3 --version v3.33.0 --api-versions admissionregistration.k8s.io/v1/MutatingAdmissionPolicy | k apply --server-side -f -`
  - Install the helm chart:
    ```
      helm install calico projectcalico/tigera-operator \
        --version v3.33.0 \
        --namespace tigera-operator \
        --set installation.cni.type=Calico \
        --set "installation.calicoNetwork.ipPools[0].cidr=10.244.0.0/16" \
        --set "installation.calicoNetwork.ipPools[0].encapsulation=VXLAN"
    ```
    - Watch deployment of the CNI resources: `watch k get pods -n calico-system`

# Upgrading CNI
  [Upgrading the helm chart](https://docs.tigera.io/calico/latest/operations/upgrading/kubernetes-upgrade#upgrading-an-installation-that-was-installed-using-helm)

  - Validate current version: `k get daemonset calico-node -n kube-system -o jsonpath='{.spec.template.spec.containers[*].image}'`
  - `helm repo update`
  - Search for the release version: `helm search repo projectcalico/tigera-operator`
  - Apply latest crd: `k apply --server-side --force-conflicts -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/v1_crd_projectcalico_org.yaml`
  - Update dry-run:
      ```
        helm upgrade calico projectcalico/tigera-operator \
          --version [LATEST_VERSION] \
          --namespace tigera-operator \
          --reuse-values \
          --dry-run
      ```
  - Upgrade the release:
      ```
        helm upgrade calico projectcalico/tigera-operator \
          --version [LATEST_VERSION] \
          --namespace tigera-operator \
          --reuse-values
      ```
  - Monitor rollout(Make sure all resources return to available): `watch k get tigerastatus`

## Metallb
I also deployed MetalLB's operator with [helm](https://metallb.universe.tf/installation/#installation-with-helm)
- Add helm repo: `helm repo add metallb https://metallb.github.io/metallb && helm repo update`
- Install the helm chart:
    ```
    helm install metallb metallb/metallb \
      --namespace metallb-system \
      --create-namespace \
      --version 0.14.8
    ```
- Install the address pool and l2 config resources:
  - Copy/create your ([metallb.yml](./LoadBalancer/metallb.yml)) file
  - Update the address pool range to match your subnet
  -Apply the config:  `k apply -f metal.yml'

- Verify pods: `k get pods -n metallb-system`
  - You should see a controller deployment and a metallb-speaker daemonset
- Verify ipaddresspools(You should see your ip pool): `k get ipaddresspools.metallb.io -n metallb-system -o yaml`
- Verify L2 config: `k get l2advertisements.metallb.io -n metallb-system -o yaml`

# Upgrading
- Update helm repo: `helm repo update`
- Search for the release version: `helm search repo metallb/metallb`
- Update dry-run:
    ```
    helm upgrade metallb metallb/metallb \
      --namespace metallb-system \
      --version [LATEST_VERSION] \
      --dry-run
    ```
- Install the update:
    ```
    helm upgrade metallb metallb/metallb \
      --namespace metallb-system \
      --version [LATEST_VERSION]
    ```
- Validate
  - Make sure pods have started: `k get pods -n metallb-system`
  - Make LoadBalancer services are not `<pending>`: `k get svc -A`
