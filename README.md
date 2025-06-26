### Initial Development On Minikube with Colima on MacOs
    colima start --memory 8 --cpu 4
    minikube start --cpus=4

    k create ns argocd
     kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

