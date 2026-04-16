0) Podesi promenljive

Ovde samo zameni PAT i po potrebi putanje.

export PROFILE="kargo-lab"
export GITHUB_USER="BorisRadenkovic"
export GITHUB_PAT="OVDE_STAVI_SVOJ_GITHUB_PAT"

export GITOPS_REPO="https://github.com/BorisRadenkovic/kargo-demo-gitops-local.git"
export APP_IMAGE_REPO="ghcr.io/borisradenkovic/kargo-demo-python-app-local"

export GITOPS_DIR="$HOME/IdeaProjects/kargo-demo-gitops-local"
export APP_DIR="$HOME/IdeaProjects/kargo-demo-python-app-local"
1) Full cleanup starog pokušaja

Ovo čisti:

zaglavljene minikube procese
kubectl port-forward
stari profil
lock fajlove
polumrtav docker container kargo-lab
pkill -9 -f "minikube" 2>/dev/null || true
pkill -9 -f "kubectl port-forward" 2>/dev/null || true

minikube stop -p "$PROFILE" 2>/dev/null || true
minikube delete -p "$PROFILE" 2>/dev/null || true

docker rm -f "$PROFILE" 2>/dev/null || true

rm -rf "$HOME/.minikube/profiles/$PROFILE"
rm -rf "$HOME/.minikube/machines/$PROFILE"
find "$HOME/.minikube" -type f -name "*.lock" -delete 2>/dev/null || true

sudo systemctl restart docker
sleep 5
docker ps -a
2) Podigni Minikube sa 4096 MB

Ovo je tvoja nova baza.

minikube start -p "$PROFILE" \
  --driver=docker \
  --cpus=2 \
  --memory=4096mb \
  --disk-size=30g

kubectl config use-context "$PROFILE"
kubectl get nodes -o wide
3) Uključi ingress
minikube addons enable ingress -p "$PROFILE"
kubectl get pods -n ingress-nginx -w

Kad vidiš ingress-nginx-controller da je Running, idi dalje.

4) Instaliraj cert-manager

Kargo ti bez ovoga pada zbog Certificate i Issuer resource-a.

helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --wait

kubectl get pods -n cert-manager
kubectl get crd | grep cert-manager.io
5) Instaliraj Argo CD
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -

kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl rollout status statefulset/argocd-application-controller -n argocd --timeout=300s
kubectl rollout status deployment/argocd-server -n argocd --timeout=300s
kubectl rollout status deployment/argocd-repo-server -n argocd --timeout=300s
kubectl rollout status deployment/argocd-applicationset-controller -n argocd --timeout=300s || true

kubectl get pods -n argocd
6) Stabilizuj Argo odmah posle install-a

Ovo je deo koji ti najviše smanjuje bagovanje.

6.1 Ugasimo ApplicationSet controller

Ne koristiš ga u ovom labu, a ume da pravi šum i troši resurse.

kubectl scale deployment argocd-applicationset-controller -n argocd --replicas=0
6.2 Povećaj timeout i git retry
kubectl patch configmap argocd-cmd-params-cm -n argocd --type merge -p '{
  "data": {
    "controller.repo.server.timeout.seconds": "180",
    "server.repo.server.timeout.seconds": "180"
  }
}'

kubectl set env deployment/argocd-repo-server -n argocd \
  ARGOCD_GIT_ATTEMPTS_COUNT=5 \
  ARGOCD_EXEC_TIMEOUT=180s
6.3 Daj Argo komponentama resurse
kubectl set resources deployment/argocd-repo-server -n argocd \
  --requests=cpu=200m,memory=512Mi \
  --limits=cpu=1000m,memory=1Gi

kubectl set resources statefulset/argocd-application-controller -n argocd \
  --requests=cpu=200m,memory=512Mi \
  --limits=cpu=1000m,memory=1Gi

kubectl set resources deployment/argocd-server -n argocd \
  --requests=cpu=100m,memory=256Mi \
  --limits=cpu=500m,memory=512Mi

kubectl set resources deployment/argocd-redis -n argocd \
  --requests=cpu=50m,memory=128Mi \
  --limits=cpu=250m,memory=256Mi
6.4 Dodaj Argo repository secret sa PAT-om

Nemoj da repo-server ide anonimno ka GitHub-u.

kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gitops-repo-credentials
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  url: ${GITOPS_REPO}
  type: git
  username: ${GITHUB_USER}
  password: ${GITHUB_PAT}
EOF
6.5 Restartuj Argo da pokupi izmene
kubectl rollout restart deployment/argocd-repo-server -n argocd
kubectl rollout restart statefulset/argocd-application-controller -n argocd
kubectl rollout restart deployment/argocd-server -n argocd
kubectl rollout restart deployment/argocd-redis -n argocd

kubectl rollout status deployment/argocd-repo-server -n argocd --timeout=300s
kubectl rollout status statefulset/argocd-application-controller -n argocd --timeout=300s
kubectl rollout status deployment/argocd-server -n argocd --timeout=300s
kubectl rollout status deployment/argocd-redis -n argocd --timeout=300s

kubectl get pods -n argocd
7) Primeni Argo Application fajlove iz tvog GitOps repoa
cd "$GITOPS_DIR"

kubectl apply -f argocd/demo-app-dev.yaml
kubectl apply -f argocd/demo-app-itg.yaml
kubectl apply -f argocd/demo-app-pro.yaml

kubectl get applications -n argocd

Po potrebi odmah uradi hard refresh:

kubectl annotate application demo-app-dev -n argocd argocd.argoproj.io/refresh=hard --overwrite
kubectl annotate application demo-app-itg -n argocd argocd.argoproj.io/refresh=hard --overwrite
kubectl annotate application demo-app-pro -n argocd argocd.argoproj.io/refresh=hard --overwrite

kubectl get applications -n argocd -w
8) Instaliraj Kargo

Ako ti apache2-utils nije tu, treba za hash password-a.

sudo apt-get update
sudo apt-get install -y apache2-utils

KARGO_PASS=$(openssl rand -base64 48 | tr -d '=+/' | head -c 32)
KARGO_HASH=$(htpasswd -bnBC 10 "" "$KARGO_PASS" | tr -d ':\n')
KARGO_SIGNING_KEY=$(openssl rand -base64 48 | tr -d '=+/' | head -c 32)

helm upgrade --install kargo oci://ghcr.io/akuity/kargo-charts/kargo \
  -n kargo \
  --create-namespace \
  --set api.adminAccount.passwordHash="$KARGO_HASH" \
  --set api.adminAccount.tokenSigningKey="$KARGO_SIGNING_KEY" \
  --wait

echo "KARGO ADMIN PASSWORD: $KARGO_PASS"

kubectl get pods -n kargo
kubectl get crd | grep kargo

Sačuvaj ovu šifru negde.

9) Primeni Kargo resurse pravilnim redosledom

Važno: nemoj ručno praviti kargo-demo namespace pre 00-project.yaml, jer si tu ranije upadao u konflikt.

9.1 Kreiraj Project
cd "$GITOPS_DIR"

kubectl apply -f kargo/00-project.yaml
kubectl get projects.kargo.akuity.io -A
kubectl get ns kargo-demo
9.2 Tek sad napravi git secret u kargo-demo
kubectl create secret generic gitops-repo-creds -n kargo-demo \
  --from-literal=repoURL="${GITOPS_REPO}" \
  --from-literal=username="${GITHUB_USER}" \
  --from-literal=password="${GITHUB_PAT}" \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl label secret gitops-repo-creds -n kargo-demo \
  kargo.akuity.io/cred-type=git --overwrite
9.3 Primeni ostatak Kargo fajlova
kubectl apply -f kargo/02-project-config.yaml
kubectl apply -f kargo/03-warehouse.yaml
kubectl apply -f kargo/04-promotion-task.yaml
kubectl apply -f kargo/05-stage-dev.yaml
kubectl apply -f kargo/06-stage-itg.yaml
kubectl apply -f kargo/07-stage-pro.yaml
9.4 Provera
kubectl get projectconfigs.kargo.akuity.io -A
kubectl get warehouses.kargo.akuity.io -n kargo-demo
kubectl get promotiontasks.kargo.akuity.io -n kargo-demo
kubectl get stages.kargo.akuity.io -n kargo-demo
kubectl get freight.kargo.akuity.io -n kargo-demo
kubectl get promotions.kargo.akuity.io -n kargo-demo
10) UI pristup
Argo CD UI
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
kubectl port-forward svc/argocd-server -n argocd 8080:443

Otvori:

https://localhost:8080
Kargo UI
kubectl port-forward svc/kargo-api -n kargo 3000:443

Otvori:

https://localhost:3000

Login:

user: admin
password: vrednost iz KARGO_PASS
11) Kad napraviš novi RC tag, gledaj ovo
kubectl get applications -n argocd -w
kubectl get freight.kargo.akuity.io -n kargo-demo -w
kubectl get promotions.kargo.akuity.io -n kargo-demo -w
kubectl get stages.kargo.akuity.io -n kargo-demo -w
12) Ako opet krene da baguje

Ovo su prve 4 dijagnostičke komande:

kubectl get pods -n argocd
kubectl get applications -n argocd
kubectl logs -n argocd deployment/argocd-repo-server --tail=80
kubectl logs -n kargo deployment/kargo-controller --tail=120
13) Najkraći redosled ako hoćeš samo “copy/paste podsvesnik”
# 1. cleanup
pkill -9 -f "minikube" 2>/dev/null || true
pkill -9 -f "kubectl port-forward" 2>/dev/null || true
minikube stop -p kargo-lab 2>/dev/null || true
minikube delete -p kargo-lab 2>/dev/null || true
docker rm -f kargo-lab 2>/dev/null || true
rm -rf ~/.minikube/profiles/kargo-lab ~/.minikube/machines/kargo-lab
find ~/.minikube -type f -name "*.lock" -delete 2>/dev/null || true
sudo systemctl restart docker
sleep 5

# 2. start minikube
minikube start -p kargo-lab --driver=docker --cpus=2 --memory=4096mb --disk-size=30g
kubectl config use-context kargo-lab
minikube addons enable ingress -p kargo-lab

# 3. cert-manager
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true --wait

# 4. argocd
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 5. argocd tuning
kubectl scale deployment argocd-applicationset-controller -n argocd --replicas=0
kubectl patch configmap argocd-cmd-params-cm -n argocd --type merge -p '{"data":{"controller.repo.server.timeout.seconds":"180","server.repo.server.timeout.seconds":"180"}}'
kubectl set env deployment/argocd-repo-server -n argocd ARGOCD_GIT_ATTEMPTS_COUNT=5 ARGOCD_EXEC_TIMEOUT=180s

# 6. argocd repo secret
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: gitops-repo-credentials
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  url: https://github.com/BorisRadenkovic/kargo-demo-gitops-local.git
  type: git
  username: BorisRadenkovic
  password: OVDE_STAVI_PAT
EOF

# 7. argocd restart
kubectl rollout restart deployment/argocd-repo-server -n argocd
kubectl rollout restart statefulset/argocd-application-controller -n argocd
kubectl rollout restart deployment/argocd-server -n argocd

# 8. apply argo apps
cd ~/IdeaProjects/kargo-demo-gitops-local
kubectl apply -f argocd/demo-app-dev.yaml
kubectl apply -f argocd/demo-app-itg.yaml
kubectl apply -f argocd/demo-app-pro.yaml

# 9. kargo
sudo apt-get update && sudo apt-get install -y apache2-utils
KARGO_PASS=$(openssl rand -base64 48 | tr -d '=+/' | head -c 32)
KARGO_HASH=$(htpasswd -bnBC 10 "" "$KARGO_PASS" | tr -d ':\n')
KARGO_SIGNING_KEY=$(openssl rand -base64 48 | tr -d '=+/' | head -c 32)
helm upgrade --install kargo oci://ghcr.io/akuity/kargo-charts/kargo \
  -n kargo --create-namespace \
  --set api.adminAccount.passwordHash="$KARGO_HASH" \
  --set api.adminAccount.tokenSigningKey="$KARGO_SIGNING_KEY" \
  --wait
echo "KARGO PASSWORD: $KARGO_PASS"

# 10. kargo project + secret + resources
kubectl apply -f kargo/00-project.yaml
kubectl create secret generic gitops-repo-creds -n kargo-demo \
  --from-literal=repoURL=https://github.com/BorisRadenkovic/kargo-demo-gitops-local.git \
  --from-literal=username=BorisRadenkovic \
  --from-literal=password=OVDE_STAVI_PAT \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl label secret gitops-repo-creds -n kargo-demo kargo.akuity.io/cred-type=git --overwrite
kubectl apply -f kargo/02-project-config.yaml
kubectl apply -f kargo/03-warehouse.yaml
kubectl apply -f kargo/04-promotion-task.yaml
kubectl apply -f kargo/05-stage-dev.yaml
kubectl apply -f kargo/06-stage-itg.yaml
kubectl apply -f kargo/07-stage-pro.yaml

Ovo ti je praktično kompletan “reset and rebuild” za tvoj lab. Sačuvaj ga lokalno kao .md ili .sh podsvesnik, jer je upravo to što ti treba kad VM krene da pravi haos.
