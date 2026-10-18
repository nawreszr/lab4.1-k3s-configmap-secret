# Lab 4.1 : Externaliser la configuration avec Kubernetes (K3s)

## 📖 Objectif du Lab
Ce laboratoire démontre comment séparer la configuration du code applicatif dans Kubernetes. 
Nous utilisons des **ConfigMaps** (pour les données non sensibles) et des **Secrets** (pour les mots de passe) afin de les injecter dans un Pod de 4 manières différentes :
1. En variables d'environnement (`envFrom`).
2. En variables d'environnement ciblées (`configMapKeyRef`, `secretKeyRef`).
3. En fichiers montés via des volumes.
4. En remplaçant un fichier de configuration complet (avec `subPath`).

## 📋 Prérequis
*   Un cluster Kubernetes (K3s) fonctionnel.
*   `kubectl` configuré sur la machine d'administration.

---

## 🚀 Guide de réalisation étape par étape

### Étape 1 : Préparer le namespace
```bash
kubectl create namespace tp-k8s
kubectl config set-context --current --namespace=tp-k8s
Étape 2 : Créer un ConfigMap
cat <<'EOF' > app-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: dev
  APP_COLOR: bleu
  LOG_LEVEL: info
EOF

kubectl apply -f app-config.yaml
kubectl get configmap app-config
kubectl describe configmap app-config
Étape 3 : Créer un Secret (Sécurité)
# Création en ligne de commande
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=appuser \
  --from-literal=DB_PASSWORD='ChangeMe-123'

# Décoder le mot de passe pour prouver que le Base64 n'est pas du chiffrement
kubectl get secret db-credentials -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo
Étape 4 :kubectl apply -f demo.yaml
kubectl rollout status deployment/demo
Déployer l'application avec les 4 modes d'injection
Étape 5 : Vérifier l'injection
# Vérifier les variables d'environnement
kubectl exec deploy/demo -- env | grep -E 'APP_|NIVEAU_|DB_'

# Vérifier les fichiers montés
kubectl exec deploy/demo -- ls /etc/app /etc/secret
kubectl exec deploy/demo -- cat /etc/app/APP_COLOR
# Vérifier les variables d'environnement
kubectl exec deploy/demo -- env | grep -E 'APP_|NIVEAU_|DB_'

# Vérifier les fichiers montés
kubectl exec deploy/demo -- ls /etc/app /etc/secret
kubectl exec deploy/demo -- cat /etc/app/APP_COLOR
Étape 6 : Modifier la configuration et observer le comportement
# Changer la valeur de "bleu" à "vert"
sed -i 's|APP_COLOR: bleu|APP_COLOR: vert|' app-config.yaml
kubectl apply -f app-config.yaml

# Vérification immédiate
kubectl exec deploy/demo -- sh -c 'echo "variable : $APP_COLOR"; echo "fichier   : $(cat /etc/app/APP_COLOR)"'

# Attendre 90 secondes que le kubelet synchronise le volume
sleep 90
kubectl exec deploy/demo -- sh -c 'echo "variable : $APP_COLOR"; echo "fichier   : $(cat /etc/app/APP_COLOR)"'
# -> Résultat attendu : la variable reste "bleu", le fichier passe à "vert"

# Redémarrer le pod pour mettre à jour la variable
kubectl rollout restart deployment/demo
kubectl rollout status deployment/demo
Étape 7 : Monter un fichier de configuration Nginx (piège du subPath)
kubectl apply -f web.yaml
kubectl rollout status deployment/web

# Test initial (v1)
kubectl run test --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://web-svc

# Modifier le ConfigMap (v1 -> v2)
sed -i 's|ConfigMap v1|ConfigMap v2|' web.yaml
kubectl apply -f web.yaml

# Observer le fichier dans le conteneur (reste en v1 à cause du subPath)
kubectl exec deploy/web -- cat /etc/nginx/conf.d/default.conf

# Redémarrer pour forcer la mise à jour
kubectl rollout restart deployment/web
kubectl rollout status deployment/web
# Test final (v2)
kubectl run test --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://web-svc
Étape 8 : Diagnostiquer un Pod en erreur (ConfigMap manquant)
# Créer un Pod qui cherche un ConfigMap inexistant
kubectl apply -f broken.yaml

# Observer l'erreur
kubectl get pod broken
kubectl describe pod broken | tail -n 8
# -> Rechercher le message : "configmap "config-manquant" not found"

# Réparer en créant le ConfigMap manquant
kubectl create configmap config-manquant --from-literal=EXEMPLE=1

# Vérifier que le Pod passe à Running tout seul
kubectl get pod broken -w
🧹 Nettoyage
kubectl delete namespace tp-k8s
kubectl config set-context --current --namespace=default
