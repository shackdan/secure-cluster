init


istioctl install --set profile=default \
  --set components.cni.enabled=true \
  --set values.istio_cni.enabled=true \
  --set values.istio_cni.chained=true \
  --set values.istio_cni.cniBinDir=/opt/cni/bin \
  --set values.istio_cni.cniConfDir=/etc/cni/net.d \
  --set values.istio_cni.excludeNamespaces={kube-system,istio-system}



  istioctl install --set profile=default \
  --set components.cni.enabled=true \
  --set values.istio_cni.enabled=true \
  --set values.istio_cni.chained=true \
  --set values.istio_cni.cniBinDir=/opt/cni/bin \
  --set values.istio_cni.cniConfDir=/etc/cni/net.d \
  --set values.istio_cni.excludeNamespaces="[kube-system,istio-system]"





  # 1. Istio Base (CRDs) - Must be installed first
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-base
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://istio-release.storage.googleapis.com/charts
    chart: base
    targetRevision: "1.26.1"
    helm:
      parameters:
      - name: defaultRevision
        value: default
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
  ignoreDifferences:
  - group: admissionregistration.k8s.io
    kind: ValidatingAdmissionWebhook
    jqPathExpressions:
    - '.webhooks[]?.clientConfig.caBundle'

---
# 2. Istio CNI - Install after base, before istiod
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-cni
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://istio-release.storage.googleapis.com/charts
    chart: cni
    targetRevision: "1.26.1"
    helm:
      parameters:
      - name: cni.chained
        value: "true"
      - name: cni.excludeNamespaces[0]
        value: kube-system
      - name: cni.excludeNamespaces[1]
        value: istio-system
      # Uncomment and adjust these if your cluster uses non-standard paths
      # - name: cni.cniBinDir
      #   value: /opt/cni/bin
      # - name: cni.cniConfDir
      #   value: /etc/cni/net.d
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true

---
# 3. Istio Control Plane (istiod) - Install after CNI
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istiod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://istio-release.storage.googleapis.com/charts
    chart: istiod
    targetRevision: "1.26.1"
    helm:
      parameters:
      - name: pilot.cni.enabled
        value: "true"  # This is crucial - tells istiod not to inject init containers
      - name: pilot.cni.provider
        value: "istio-cni"
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true
  ignoreDifferences:
  - group: admissionregistration.k8s.io
    kind: ValidatingAdmissionWebhook
    jqPathExpressions:
    - '.webhooks[]?.clientConfig.caBundle'

---
# 4. Optional: Istio Gateway (Ingress)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-gateway
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://istio-release.storage.googleapis.com/charts
    chart: gateway
    targetRevision: "1.26.1"
    helm:
      parameters:
      - name: service.type
        value: LoadBalancer
      # Customize the gateway as needed
      - name: replicaCount
        value: "2"
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-ingress
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true

---
# Alternative: Single ApplicationSet for all Istio components
# Use this instead of the individual applications above if you prefer
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: istio-stack
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - name: istio-base
        chart: base
        namespace: istio-system
        priority: "100"  # Install first
        values: |
          defaultRevision: default
      - name: istio-cni
        chart: cni
        namespace: istio-system
        priority: "200"  # Install second
        values: |
          cni:
            chained: true
            excludeNamespaces:
              - kube-system
              - istio-system
      - name: istiod
        chart: istiod
        namespace: istio-system
        priority: "300"  # Install third
        values: |
          pilot:
            cni:
              enabled: true
              provider: istio-cni
      - name: istio-gateway
        chart: gateway
        namespace: istio-ingress
        priority: "400"  # Install last
        values: |
          service:
            type: LoadBalancer
          replicaCount: 2
  template:
    metadata:
      name: '{{name}}'
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: default
      source:
        repoURL: https://istio-release.storage.googleapis.com/charts
        chart: '{{chart}}'
        targetRevision: "1.26.1"
        helm:
          values: '{{values}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
        - ServerSideApply=true
      ignoreDifferences:
      - group: admissionregistration.k8s.io
        kind: ValidatingAdmissionWebhook
        jqPathExpressions:
        - '.webhooks[]?.clientConfig.caBundle'