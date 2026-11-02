# Guide GitOps : Déploiement d'Ansible AWX & ARA sur K3s (Dev) et EKS (Prod)

Ce document décrit l'architecture et la procédure de déploiement d'**Ansible AWX** et d'**ARA (Ansible Run Analysis)** en mode GitOps en utilisant **ArgoCD** et **Kustomize**.

---

## 🏗️ Architecture du Répertoire GitOps

Pour gérer proprement la mutualisation du code et les spécificités de chaque environnement (**K3s** pour le développement et **EKS** pour la production), nous utilisons une structure Kustomize basée sur une `base` commune et des `overlays`.

```text
gitops-infra/
└── apps/
    ├── awx/
    │   ├── base/
    │   │   ├── kustomization.yaml
    │   │   └── awx-instance.yaml
    │   └── overlays/
    │       ├── dev-k3s/
    │       │   └── kustomization.yaml
    │       └── prod-eks/
    │           └── kustomization.yaml
    └── ara/
        ├── base/
        │   ├── kustomization.yaml
        │   ├── ara-deployment.yaml
        │   └── ara-service.yaml
        └── overlays/
            ├── dev-k3s/
            │   └── kustomization.yaml
            └── prod-eks/
                └── kustomization.yaml
```

---

## 🤖 Partie 1 : Déploiement d'Ansible AWX via ArgoCD

### 1. La Base Commune (`apps/awx/base/`)

#### Fichier `apps/awx/base/kustomization.yaml`
Ce fichier charge automatiquement l'**AWX Operator** via son Chart Helm officiel et déclare la ressource de base.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - awx-instance.yaml

helmCharts:
  - name: awx-operator
    repo: https://ansible-community.github.io/awx-operator-helm/
    version: 2.19.1
    releaseName: my-awx-operator
    namespace: awx
```

#### Fichier `apps/awx/base/awx-instance.yaml`
```yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  admin_user: admin
```

### 2. Les Environnements (`apps/awx/overlays/`)

#### 🛠️ Overlay Dev - K3s (`apps/awx/overlays/dev-k3s/kustomization.yaml`)
Pour le cluster de développement, nous utilisons la base de données intégrée (PostgreSQL conteneurisé), le stockage local K3s et l'Ingress Traefik natif.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - target:
      kind: AWX
      name: awx
    patch: |-
      - op: replace
        path: /spec
        value:
          postgres_configuration_secret: awx-postgres-configuration
          storage_class: local-path
          ingress_type: ingress
          ingress_ingress_class_name: traefik
          ingress_hosts:
            - hostname: awx.dev.local
```

#### 🚀 Overlay Prod - EKS (`apps/awx/overlays/prod-eks/kustomization.yaml`)
Pour la production, AWX s'appuie sur une base de données managée externe AWS RDS, le CSI driver Amazon EBS (`ebs-sc`) et un AWS Application Load Balancer (ALB).

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - target:
      kind: AWX
      name: awx
    patch: |-
      - op: replace
        path: /spec
        value:
          postgres_configuration_secret: rds-postgres-secret
          storage_class: ebs-sc
          ingress_type: ingress
          ingress_ingress_class_name: alb
          ingress_annotations:
            alb.ingress.kubernetes.io/scheme: internet-facing
            alb.ingress.kubernetes.io/target-type: ip
          ingress_hosts:
            - hostname: awx.votreentreprise.com
```

*Note : Pensez à déployer au préalable le Secret Kubernetes `rds-postgres-secret` contenant les identifiants de votre instance RDS dans le namespace `awx` du cluster EKS.*

---

## 📊 Partie 2 : Intégration d'ARA (Ansible Run Analysis)

ARA intercepte les résultats d'exécution d'Ansible (via un *callback plugin*) et les enregistre dans une base de données accessible via un serveur web de reporting.

### 1. La Base Commune d'ARA (`apps/ara/base/`)

#### Fichier `apps/ara/base/ara-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ara-server
  namespace: ara
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ara-server
  template:
    metadata:
      labels:
        app: ara-server
    spec:
      containers:
        - name: ara-server
          image: ararecordsansible/ara-server:latest
          ports:
            - containerPort: 8000
          env:
            - name: ARA_ALLOWED_HOSTS
              value: '["*"]'
```

#### Fichier `apps/ara/base/ara-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ara-service
  namespace: ara
spec:
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    app: ara-server
```

#### Fichier `apps/ara/base/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ara-deployment.yaml
  - ara-service.yaml
```

### 2. Les Ingress pour ARA (`apps/ara/overlays/`)

#### 🛠️ Ingress ARA - K3s (`apps/ara/overlays/dev-k3s/kustomization.yaml`)
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
patches:
  - patch: |-
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      metadata:
        name: ara-ingress
        namespace: ara
        annotations:
          kubernetes.io/ingress.class: traefik
      spec:
        rules:
          - host: ara.dev.local
            http:
              paths:
                - path: /
                  pathType: Prefix
                  backend:
                    service:
                      name: ara-service
                      port:
                        number: 8000
    target:
      kind: Deployment # Kustomize requiert une cible fictive ou l'ajout direct du manifeste d'ingress
```
*(Alternative recommandée : créer un fichier `ingress.yaml` dédié dans l'overlay et l'ajouter dans la liste des `resources`).*

---

## 🚀 Partie 3 : Déclaration des Applications dans ArgoCD

Pour lier votre dépôt Git à vos clusters, appliquez les manifestes d'applications suivants directement au sein de votre instance centrale ArgoCD.

### Application AWX - Environnement Dev (K3s)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: awx-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/votre-org/gitops-infra.git'
    targetRevision: HEAD
    path: apps/awx/overlays/dev-k3s
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: awx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Application ARA - Environnement Dev (K3s)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ara-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/votre-org/gitops-infra.git'
    targetRevision: HEAD
    path: apps/ara/overlays/dev-k3s
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: ara
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## 🔧 Connexion entre AWX et ARA

Pour qu'AWX envoie ses logs de rapports vers votre instance ARA, vous devez configurer le callback ARA dans l'environnement d'exécution (Execution Environment - EE) d'AWX.

1. **Dans vos playbooks Ansible (gérés par AWX) :** Ajoutez la variable d'environnement ou configurez votre fichier `ansible.cfg` pour pointer vers le serveur ARA interne au cluster :
   ```ini
   [defaults]
   callback_plugins = /usr/local/lib/python3.9/site-packages/ara/plugins/callback
   
   [callback_ara]
   api_client = http
   api_server = http://ara-service.ara.svc.cluster.local:8000
   ```
2. **Via les variables de Job Template sur AWX :** Vous pouvez aussi injecter les variables d'environnement directement lors du lancement d'un job sur AWX :
   * `ANSIBLE_LOAD_CALLBACK_PLUGINS: "true"`
   * `ANSIBLE_STDOUT_CALLBACK: "ara"`
   * `ARA_API_SERVER: "http://ara-service.ara.svc.cluster.local:8000"`
