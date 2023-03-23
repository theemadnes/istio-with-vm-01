# istio-with-vm-01
testing Istio on GKE with GCE VM to test if locality LB works from VM -> GKE

```
curl -L https://istio.io/downloadIstio | sh -

export PATH="$PATH:/home/user/istio-with-vm-01/istio-1.17.1/bin"

istioctl install --set profile=default -y

# need to edit replicas for ingress gateway as default is 1

kubectl -n istio-system edit hpa istio-ingressgateway

$ kubectl get pods -A
NAMESPACE      NAME                                                       READY   STATUS              RESTARTS   AGE
istio-system   istio-ingressgateway-7954975d69-2gfc7                      1/1     Running             0          6s
istio-system   istio-ingressgateway-7954975d69-hqnrq                      0/1     ContainerCreating   0          6s
istio-system   istio-ingressgateway-7954975d69-k6bv6                      1/1     Running             0          2m7s

# get whereami up and running and test access from outside of cluster 
kubectl create namespace whereami
kubectl label namespace whereami istio-injection=enabled

git clone https://github.com/GoogleCloudPlatform/kubernetes-engine-samples.git

kubectl -n whereami apply -k kubernetes-engine-samples/whereami/k8s
kubectl apply -f whereami-gateway.yaml
kubectl apply -f whereami-vs.yaml 

$ curl --header "Host: whereami.example.com" http://35.239.31.157
{
  "cluster_name": "istio-test-01",
  "gce_instance_id": "6815765797943919684",
  "gce_service_account": "spanner-reporter-01.svc.id.goog",
  "host_header": "whereami.example.com",
  "metadata": "frontend",
  "node_name": "gke-istio-test-01-pool-1-cb1d2e26-6fxb",
  "pod_ip": "10.72.0.3",
  "pod_name": "whereami-75b96b4f95-tgql8",
  "pod_name_emoji": "👿",
  "pod_namespace": "whereami",
  "pod_service_account": "whereami",
  "project_id": "spanner-reporter-01",
  "timestamp": "2023-03-21T19:21:59",
  "zone": "us-central1-c"
}




### created VM and now getting whereami on it and running locally 
sudo apt update
sudo apt upgrade
# install git
sudo apt install git
# install pip3 & venv
sudo apt-get install python3-venv
sudo apt-get install python3-pip

# prep and run
git clone https://github.com/GoogleCloudPlatform/kubernetes-engine-samples.git
python3 -m venv kubernetes-engine-samples/whereami/
cd kubernetes-engine-samples/whereami/
source bin/activate
pip install -r requirements.txt
python app.py

# following https://istio.io/latest/docs/setup/install/virtual-machine/
export VM_APP=whereami-vm
export VM_NAMESPACE=whereami-vm
export WORK_DIR=whereami-vm
export SERVICE_ACCOUNT=whereami-vm
export CLUSTER_NETWORK=""
export VM_NETWORK=""
export CLUSTER="Kubernetes"

mkdir -p "${WORK_DIR}"

istio-1.17.1/samples/multicluster/gen-eastwest-gateway.sh --single-cluster | istioctl install -y -f -
kubectl apply -n istio-system -f istio-1.17.1/samples/multicluster/expose-istiod.yaml

kubectl create namespace "${VM_NAMESPACE}"

kubectl create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"

cat <<EOF > ./vm-cluster.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: "${CLUSTER}"
      network: "${CLUSTER_NETWORK}"
EOF


istioctl install -f vm-cluster.yaml --set values.pilot.env.PILOT_ENABLE_WORKLOAD_ENTRY_AUTOREGISTRATION=true --set values.pilot.env.PILOT_ENABLE_WORKLOAD_ENTRY_HEALTHCHECKS=true



cat <<EOF > workloadgroup.yaml
apiVersion: networking.istio.io/v1alpha3
kind: WorkloadGroup
metadata:
  name: "${VM_APP}"
  namespace: "${VM_NAMESPACE}"
spec:
  metadata:
    labels:
      app: "${VM_APP}"
  template:
    serviceAccount: "${SERVICE_ACCOUNT}"
    network: "${NETWORK}"
  probe:
    periodSeconds: 5
    initialDelaySeconds: 1
    httpGet:
      port: 8080
      path: /healthz
EOF

kubectl --namespace "${VM_NAMESPACE}" apply -f workloadgroup.yaml

istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --autoregister

#### copying files from workstation
gcloud compute scp --project=spanner-reporter-01 --zone=us-central1-c whereami-vm/* mesh-vm-c-1:/home/admin/

#### from the VM

sudo mkdir -p /etc/certs
sudo cp "${HOME}"/root-cert.pem /etc/certs/root-cert.pem


sudo  mkdir -p /var/run/secrets/tokens
sudo cp "${HOME}"/istio-token /var/run/secrets/tokens/istio-token

curl -LO https://storage.googleapis.com/istio-release/releases/1.17.1/deb/istio-sidecar.deb
sudo dpkg -i istio-sidecar.deb

sudo cp "${HOME}"/cluster.env /var/lib/istio/envoy/cluster.env
sudo cp "${HOME}"/mesh.yaml /etc/istio/config/mesh
sudo sh -c 'cat $(eval echo ~$SUDO_USER)/hosts >> /etc/hosts'
sudo mkdir -p /etc/istio/proxy
sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
sudo systemctl start istio


### problem - VM can't talk to external LBs 
### also cleaned up whereami SVC to make it ClusterIP

### from workstation
kubectl apply -f destination_rule.yaml
```

