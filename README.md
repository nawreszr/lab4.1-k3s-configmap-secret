# Lab 4.1 : Externaliser la configuration (ConfigMaps & Secrets)

## Description
Ce lab démontre comment séparer la configuration du code applicatif en utilisant des ConfigMaps et des Secrets dans Kubernetes. Il explore les 4 modes d'injection (variables, `envFrom`, `configMapKeyRef`, `secretKeyRef`) et les montages de volumes.

## Fichiers
- `app-config.yaml` : ConfigMap contenant les variables d'environnement.
- `demo.yaml` : Deployment démontrant les 4 modes d'injection.
- `web.yaml` : Déploiement Nginx utilisant un ConfigMap monté via `subPath`.
- `broken.yaml` : Pod provoquant une erreur `CreateContainerConfigError` pour diagnostic.
- `secret.example.yaml` : Modèle pour le Secret (ne contient pas de vrai mot de passe).

## Utilisation
1. Créer le namespace : `kubectl create namespace tp-k8s`
2. Créer le Secret réel (à partir du modèle ou en ligne de commande) :
   `kubectl create secret generic db-credentials --from-literal=DB_USER=appuser --from-literal=DB_PASSWORD='ChangeMe-123'`
3. Appliquer les manifestes : `kubectl apply -f app-config.yaml -f demo.yaml -f web.yaml`
