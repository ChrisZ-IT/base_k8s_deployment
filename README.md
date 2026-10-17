# base_k8s_deployment
base k8s config for networking and LB for on prem/homelab use.
Assumes you installed k8s via kubeadm.

## Installing Calico CNI
 I went with calico for my K8s CNI and I am using the [helm deployment](https://docs.tigera.io/calico/latest/getting-started/kubernetes/helm) for this. Use official docs over this one.

  - Add the helm repo: `helm repo add projectcalico https://docs.tigera.io/calico/charts && helm repo update`
  - Search for the release version: `helm search repo projectcalico/tigera-operator`
  - Install the calico CRDs: `helm template calico-crds projectcalico/projectcalico.org.v3 --version [CHART_VERSION] --api-versions admissionregistration.k8s.io/v1/MutatingAdmissionPolicy | k apply --server-side -f -`
  - Install the helm chart:
    ```
      helm install calico projectcalico/tigera-operator \
        --version [CHART_VERSION] \
        --namespace tigera-operator \
        --set installation.cni.type=Calico \
        --set "installation.calicoNetwork.ipPools[0].cidr=10.244.0.0/16" \
        --set "installation.calicoNetwork.ipPools[0].encapsulation=VXLAN"
    ```
    - Watch deployment of the CNI resources: `watch k get pods -n calico-system`

# Upgrading Calico CNI
  [Upgrading the helm chart](https://docs.tigera.io/calico/latest/operations/upgrading/kubernetes-upgrade#upgrading-an-installation-that-was-installed-using-helm)

  - Validate current version: `k get daemonset calico-node -n kube-system -o jsonpath='{.spec.template.spec.containers[*].image}'`
  - `helm repo update`
  - Search for the release version: `helm search repo projectcalico/tigera-operator`
  - Apply latest crd: `k apply --server-side --force-conflicts -f https://raw.githubusercontent.com/projectcalico/calico/[CHART_VERSION]/manifests/v1_crd_projectcalico_org.yaml`
  - Update dry-run:
      ```
        helm upgrade calico projectcalico/tigera-operator \
          --version [CHART_VERSION] \
          --namespace tigera-operator \
          --reuse-values \
          --dry-run
      ```
  - Upgrade the release:
      ```
        helm upgrade calico projectcalico/tigera-operator \
          --version [CHART_VERSION] \
          --namespace tigera-operator \
          --reuse-values
      ```
  - Monitor rollout(Make sure all resources return to available): `watch k get tigerastatus`

## Installing Metallb
I also deployed [MetalLB's operator with helm](https://metallb.universe.tf/installation/#installation-with-helm)
- Add helm repo: `helm repo add metallb https://metallb.github.io/metallb && helm repo update`
- Search for the release version: `helm search repo metallb/metallb`
- Install the helm chart:
    ```
    helm install metallb metallb/metallb \
      --namespace metallb-system \
      --create-namespace \
      --version [CHART_VERSION]
    ```
- Install the address pool and l2 config resources:
  - Copy/create your ([metallb.yml](./LoadBalancer/metallb.yml)) file
  - Update the address pool range to match your subnet
  -Apply the config:  `k apply -f metal.yml'

- Verify pods: `k get pods -n metallb-system`
  - You should see a controller deployment and a metallb-speaker daemonset
- Verify ipaddresspools(You should see your ip pool): `k get ipaddresspools.metallb.io -n metallb-system -o yaml`
- Verify L2 config: `k get l2advertisements.metallb.io -n metallb-system -o yaml`

# Upgrading Metallb
- Update helm repo: `helm repo update`
- Search for the release version: `helm search repo metallb/metallb`
- Update dry-run:
    ```
    helm upgrade metallb metallb/metallb \
      --namespace metallb-system \
      --version [CHART_VERSION] \
      --dry-run
    ```
- Install the update:
    ```
    helm upgrade metallb metallb/metallb \
      --namespace metallb-system \
      --version [CHART_VERSION]
    ```
- Validate
  - Make sure pods have started: `k get pods -n metallb-system`
  - Make LoadBalancer services are not `<pending>`: `k get svc -A`

## Installing Istio
Since this is my homelab Im skipping doing the canary deployment of istio.

Installed [isto ingress with helm](https://istio.io/latest/docs/setup/install/helm/)

- Add the isto helm repo: `helm repo add istio https://blob.istio.io/istio-release/charts && helm repo update`
- Search base istio chart: `helm search repo istio/base` to get the version number for below `[CHART_VERSION]`
- install base istio:
  ```
    helm install istio-base istio/base -n istio-system \
      --set defaultRevision=default \
      --version [CHART_VERSION] \
      --create-namespace
  ```
- Install istiod: `helm install istiod istio/istiod -n istio-system --wait`
- validate: `helm ls -n istio-system`
- Install Istio-gateway:
  - install: `helm install istio-ingress istio/gateway -n istio-ingress --create-namespace --wait`
- Add namespace label as to the name spaces that need it: `k label namespace <namespace name> istio-injection=enabled`

# Upgrading istio
  Note: Since this single control plane deployment your istio services will experience a brief outage. If you want to avoid that switch to a canary deployment of the istio control planes and services. I figure its easy enough to just use a `helm rollback` if I have issues with an upgrade.

- Update helm repo: `helm repo update`
- Search for the release version: `helm search repo istio/base`
- Validate Istio base update(dry-run):
    ```
    helm upgrade istio-base istio/base \
      -n istio-system \
      --set defaultRevision=default \
      --version [CHART_VERSION] \
      --dry-run
    ```
- Install the update:
    ```
    helm upgrade istio-base istio/base \
      -n istio-system \
      --set defaultRevision=default \
      --version [CHART_VERSION]
    ```

- Validate istiod(dry-run):
  ```
    helm upgrade istiod istio/istiod \
      -n istio-system \
      --set revision=default \
      --version [CHART_VERSION] \
      --wait \
      --dry-run
  ```
- Validate istio-ingress(dry-run):
  ```
    helm upgrade istio-ingress istio/gateway \
      -n istio-ingress \
      --set revision=default \
      --version [CHART_VERSION] \
      --dry-run
  ```
- Install istio-ingress-canary:
  ```
    helm upgrade istio-ingress istio/gateway \
      -n istio-ingress \
      --set revision=default \
      --version [CHART_VERSION]
  ```
- Validate istio-ingress(dry-run):
  ```
    helm upgrade istio-ingress istio/gateway \
      -n istio-ingress \
      --version [CHART_VERSION] \
      --dry-run
  ```
- Install istio-ingress:
  ```
    helm upgrade istio-ingress istio/gateway \
      -n istio-ingress \
      --version [CHART_VERSION]
  ```

- Validate istio pods:
  - `k get pods -n istio-system`
  - `k get pods -n istio-ingress`

- Restart deployments using istio: `k rollout restart deployments -n <your-application-namespace>`
