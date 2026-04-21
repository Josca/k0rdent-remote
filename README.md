# k0rdent Remote Cluster Example

Remote (SSH) cluster provisioning example with Cilium CNI provider.

## Verified setup
Prepare 3 Linux machines (tested with Ubuntu 24.04) in the same network.
- 1. k0rdent - running k0s-1.35.3 with working default storageclass (e.g. openebs)
- 2. worker1
- 3. worker2

## Prepare SSJ access to workers
Ensure SSH root access to machines worker1 and worker2:
~~~bash
ssh-keygen -f ~/.ssh/idk0r # generate extra ssh key-pair for the case
~~~

~~~bash
ssh-copy-id -i ~/.ssh/idk0r.pub root@worker1
ssh-copy-id -i ~/.ssh/idk0r.pub root@worker2
~~~

## Install k0rdent mothership
In k0rdent machine install kcm 1.8.0
~~~bash
helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm --version 1.8.0 -n kcm-system --create-namespace
~~~

## Setup Remote Credential objects
~~~bash
helm install remote-credential oci://ghcr.io/k0rdent/catalog/charts/remote-credential -n kcm-system
REMOTE_SSH_KEY_B64=$(cat ~/.ssh/idk0r | openssl base64 -A) # setup secret
kubectl patch secret remote-ssh-key -n kcm-system -p='{"data":{"value":"'$REMOTE_SSH_KEY_B64'"}}'
~~~

## Install Cilium k0rdent service template
~~~bash
helm upgrade --install cilium oci://ghcr.io/k0rdent/catalog/charts/kgst --set "chart=cilium:1.19.0" -n kcm-system

# verify service template
kubectl get servicetemplates -A | grep cilium
# NAMESPACE    NAME                            VALID
# kcm-system   cilium-1-19-0                   true
~~~

## Prepare ClusterDeployment config
Create `remote-cld-cilium.yaml` file, don't forget to set IP addresses properly (worker1-ip, worker2-ip):
~~~yaml
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ClusterDeployment
metadata:
  name: remote
  namespace: kcm-system
  labels:
    type: remote
spec:
  template: remote-cluster-1-0-22
  credential: remote-credential
  propagateCredentials: false
  config:
    controlPlaneNumber: 1 # Using simple setup 1 master, 2 workers
    k0smotron:
      service:
        type: NodePort # Simple setup with no master loadbalancer (nodes run in a single network)
    machines:
    - name: worker1
      address: <worker1-ip>
      user: root # The user must have root permissions
      port: 22
      labels:
        node-role.kubernetes.io/system-worker: "true"
    - name: worker2
      address: <worker2-ip>
      user: root # The user must have root permissions
      port: 22
      labels:
        node-role.kubernetes.io/system-worker: "true"

    k0s:
      version: v1.35.3+k0s.0
      network: # prepare for cilium
        calico: null
        provider: custom
        kubeProxy:
          disabled: true

  serviceSpec:
    services:
    - template: cilium-1-19-0
      name: cilium
      namespace: cilium
      values: |
        cilium:
          cluster:
            name: cilium
          hubble:
            tls:
              enabled: false
            auto:
              method: helm
              certManagerIssuerRef: {}
            ui:
              enabled: false
              ingress:
                enabled: false
            relay:
              enabled: false
          ipv4:
            enabled: true
          ipv6:
            enabled: false
          envoy:
            enabled: false
          egressGateway:
            enabled: false
          kubeProxyReplacement: "true"
          serviceAccounts:
            cilium:
              name: cilium
            operator:
              name: cilium-operator
          localRedirectPolicy: true
          ipam:
            mode: cluster-pool
            operator:
              clusterPoolIPv4PodCIDRList:
              - "192.168.224.0/20"
              - "192.168.210.0/20"
              clusterPoolIPv6PodCIDRList:
              - "fd00::/104"
          tunnelProtocol: geneve
          k8sServiceHost: "{{ .Cluster.spec.controlPlaneEndpoint.host }}"
          k8sServicePort: "{{ .Cluster.spec.controlPlaneEndpoint.port }}"
~~~

## Deploy ClusterDeployment
~~~bash
kubectl apply -f remote-cld-cilium.yaml
~~~
Wait for stable status:
~~~bash
kubectl get cld -A
# NAMESPACE    NAME     READY   SERVICES   TEMPLATE                MESSAGES          AGE
# kcm-system   remote   True    1/1        remote-cluster-1-0-22   Object is ready   15m
~~~

## Access child cluster (from k0rdent node)
~~~bash
kubectl get secret remote-kubeconfig -n kcm-system -o=jsonpath={.data.value} | base64 -d > kcfg_remote
KUBECONFIG=kcfg_remote kubectl get pod -A
# NAMESPACE        NAME                                     READY   STATUS    RESTARTS   AGE
# cilium           cilium-4kj45                             1/1     Running   0          16m
# cilium           cilium-operator-76dc9b8948-p55h5         1/1     Running   0          17m
# cilium           cilium-operator-76dc9b8948-xvp7q         1/1     Running   0          17m
# cilium           cilium-t22pq                             1/1     Running   0          16m
# kube-system      coredns-5f55cfb9f6-5xk9f                 1/1     Running   0          16m
# kube-system      coredns-5f55cfb9f6-f6kkl                 1/1     Running   0          16m
# kube-system      konnectivity-agent-kvrcz                 1/1     Running   0          16m
# kube-system      konnectivity-agent-qn6xv                 1/1     Running   0          16m
# kube-system      metrics-server-df68c566c-m8jmm           1/1     Running   0          17m
# projectsveltos   sveltos-agent-manager-857d5594fc-7gj9p   1/1     Running   0          17m
~~~

## Troubleshooting

### Control plane pod `watcher` error
Issue: Getting error in k0rdent pod `kmc-remote-0`: `level=error msg="Failed to create watcher" component=applier-manager error="too many open files"`.

Solution: Updating Linux resource limits:
~~~bash
ulimit -n # 1024 increased to 65535
sysctl fs.inotify.max_user_watches # 250035 increased to 524288
sysctl fs.inotify.max_user_instances # 128 increased to 512
~~~
