### Initial Development On Minikube with Colima on MacOs
    colima start --memory 8 --cpu 4
    minikube start --cpus=4
    istioctl install --set profile=default --set values.global.platform=minikube  --set values.istio_cni.enabled=true \
  --set values.istio_cni.chained=true

    k create ns argocd
     kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

###
kubectl get applications.argoproj.io -n argocd -o name | xargs -n1 kubectl delete -n argocd

### Istio


yaml:
global:
  istioNamespace: istio-system
  istio_cni:
    enabled: true
    chained: true
  caAddress: ""
  trustDomain: "cluster.local"
  platform: minikube

meshConfig:
  defaultConfig:
    proxyMetadata:
      ISTIO_META_CNI_ENABLED: "true"

pilot:
  env:
    EXTERNAL_ISTIOD: false


cat <<EOF > istio-default.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components:
    cni:
      namespace: istio-system
      enabled: true
EOF
istioctl install -f istio-cni.yaml -y


kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components:
    cni:
      namespace: istio-system
      enabled: true
EOF