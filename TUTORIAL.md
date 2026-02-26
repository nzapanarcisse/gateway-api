# Atelier Pratique : Déployer Odoo sur Kubernetes avec la Gateway API

Bienvenue dans cet atelier pratique ! En tant que futurs experts du Cloud Native, vous allez découvrir comment exposer des applications de manière moderne et standardisée sur Kubernetes. Nous allons laisser de côté l'ancien `Ingress` pour nous plonger dans son successeur : la **Gateway API**.

## Introduction : Pourquoi abandonner l'Ingress pour la Gateway API ?

L'objet `Ingress` a été la première façon de gérer l'accès HTTP externe à des services dans Kubernetes. Cependant, il présentait des limites :
- **Trop simple** : Pour des besoins avancés (TLS, poids de trafic, etc.), il fallait utiliser des annotations propriétaires (`nginx.ingress.kubernetes.io/rewrite-target`), ce qui rendait les déploiements dépendants d'un seul fournisseur (Nginx, Traefik, etc.).
- **Peu flexible** : Il n'y avait pas de séparation claire des rôles. Le développeur qui déployait son application devait aussi gérer l'infrastructure réseau, ce qui n'est pas idéal dans les grandes organisations.

La **Gateway API** est une nouvelle spécification qui vise à résoudre ces problèmes. Elle est :
- **Standardisée et Expressive** : Les fonctionnalités avancées sont intégrées dans l'API elle-même, plus besoin d'annotations.
- **Orientée Rôle** : Elle sépare les responsabilités entre :
    1.  **L'Admin de l'Infrastructure** (qui installe les `GatewayClass`).
    2.  **L'Opérateur du Cluster** (qui déploie et configure les `Gateway`).
    3.  **Le Développeur** (qui définit les règles de routage pour son application via les `HTTPRoute`).
- **Extensible** : Elle peut gérer plus que du trafic HTTP (TCP, UDP, gRPC) à l'avenir.

## Étape 0 : Prérequis et Installation Commune

Avant de comparer nos deux solutions, nous devons préparer notre cluster.

### 1. Valider le Cluster
Assurez-vous que votre cluster Kubernetes (lancé via le `Vagrantfile`) est fonctionnel.

```bash
# Cette commande doit lister vos nœuds (master, worker1).
kubectl get nodes
```

### 2. Installer les CRDs de la Gateway API
Les objets `Gateway` et `HTTPRoute` ne sont pas inclus par défaut dans Kubernetes. Nous devons ajouter leur définition (CRD - Custom Resource Definition) au cluster.

```bash
# Nous appliquons le fichier de configuration standard fourni par la communauté Kubernetes.
# Cela étend l'API de notre cluster avec les nouveaux types d'objets.
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

### 3. Installer et Configurer MetalLB
Dans un environnement Cloud (AWS, GCP, Azure), demander un `Service` de type `LoadBalancer` provisionne automatiquement une IP publique. En local, ce n'est pas le cas. MetalLB simule ce comportement.

**Installation :**
```bash
# On applique le manifeste officiel de MetalLB.
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

Attendez que les pods MetalLB soient lancés dans le namespace `metallb-system` :
```bash
kubectl get pods -n metallb-system -w
```

**Configuration :**
Nous allons dire à MetalLB quelle plage d'adresses IP il peut utiliser. Créez un fichier `metallb-config.yaml`.
*Note : La plage IP `192.168.56.100-192.168.56.150` est souvent compatible avec les réseaux privés de Vagrant/VirtualBox.*

```yaml
# metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.56.100-192.168.56.150
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```

Appliquez cette configuration :
```bash
# On crée le pool d'IPs et on dit à MetalLB de l'annoncer sur le réseau (mode Layer 2).
kubectl apply -f metallb-config.yaml
```

Notre cluster est prêt ! Passons maintenant au déploiement de notre application Odoo.

---

## Cas A : Implémentation avec Traefik

Traefik est un reverse-proxy et load-balancer très populaire dans l'écosystème Cloud Native.

### 1. Installation de Traefik
Nous utiliserons Helm, le gestionnaire de paquets pour Kubernetes.

```bash
# Ajouter le dépôt Helm de Traefik
helm repo add traefik https://helm.traefik.io/traefik

# Mettre à jour les informations du dépôt
helm repo update

# Installer Traefik en activant le support pour la Gateway API
helm install traefik traefik/traefik \
  --set providers.kubernetesGateway.enabled=true \
  --set experimental.kubernetesGateway.enabled=true
```
*Note : La deuxième ligne `experimental.kubernetesGateway.enabled` est nécessaire pour les versions actuelles de Traefik, cela pourrait changer.*

### 2. Déploiement d'Odoo et PostgreSQL
Nous allons créer tous les fichiers dans un dossier virtuel `/traefik-odoo`.

**Fichiers de déploiement :**

```yaml
# /traefik-odoo/postgres-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
stringData:
  POSTGRES_USER: odoo
  POSTGRES_PASSWORD: mysecretpassword
  POSTGRES_DB: postgres
```

```yaml
# /traefik-odoo/postgres-statefulset.yaml
# Pour une base de données, un StatefulSet est plus approprié qu'un Deployment.
# Il garantit une identité réseau et un stockage stables.
apiVersion: v1
kind: Service
metadata:
  name: postgres-service # Odoo se connecte à ce nom de service
  labels:
    app: postgres
spec:
  ports:
  - port: 5432
    name: postgres
  clusterIP: None # Rend le service "headless", nécessaire pour un StatefulSet
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres-service"
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        envFrom:
          - secretRef:
              name: postgres-secret
        ports:
        - containerPort: 5432
          name: postgres
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: standard # Assurez-vous que cette StorageClass existe dans votre cluster (ex: 'standard' pour minikube/kind)
      resources:
        requests:
          storage: 1Gi
```

```yaml
# /traefik-odoo/odoo-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: odoo-config
data:
  HOST: postgres-service
  USER: odoo
  PASSWORD: mysecretpassword
```

```yaml
# /traefik-odoo/odoo-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: odoo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: odoo
  template:
    metadata:
      labels:
        app: odoo
    spec:
      containers:
        - name: odoo
          image: odoo:16.0
          envFrom:
            - configMapRef:
                name: odoo-config
          ports:
            - containerPort: 8069
---
apiVersion: v1
kind: Service
metadata:
  name: odoo-service
spec:
  selector:
    app: odoo
  ports:
    - protocol: TCP
      port: 8069
      targetPort: 8069
```

**Déployez tous ces composants :**
```bash
# Créez les fichiers ci-dessus et appliquez-les
kubectl delete -f /traefik-odoo/postgres-secret.yaml
kubectl delete -f /traefik-odoo/postgres-pvc.yaml
kubectl delete -f /traefik-odoo/postgres-deployment.yaml
kubectl delete -f /traefik-odoo/odoo-configmap.yaml
kubectl delete -f /traefik-odoo/odoo-deployment.yaml
```

### 3. Configuration de la Gateway API (Traefik)

**Fichiers de Gateway :**

```yaml
# /traefik-odoo/gateway-class.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: traefik-gateway-class
spec:
  controllerName: traefik.io/gateway-controller
```

```yaml
# /traefik-odoo/gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: traefik-gateway
spec:
  gatewayClassName: traefik-gateway-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Same
```

```yaml
# /traefik-odoo/http-route.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: odoo-http-route
spec:
  parentRefs:
  - name: traefik-gateway
  hostnames: ["odoo.traefik.local"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: odoo-service
      port: 8069
```

**Appliquez ces configurations :**
```bash
kubectl apply -f /traefik-odoo/gateway-class.yaml
kubectl apply -f /traefik-odoo/gateway.yaml
kubectl apply -f /traefik-odoo/http-route.yaml
```

### 4. Accès en HTTP

1.  **Récupérez l'IP du LoadBalancer :**
    La `Gateway` a demandé un LoadBalancer. MetalLB lui a fourni une IP.

    ```bash
    kubectl get svc traefik
    ```
    Vous verrez une `EXTERNAL-IP`. Copiez-la.

2.  **Modifiez votre fichier `hosts` :**
    Pour que votre machine locale sache que `odoo.traefik.local` pointe vers cette IP.

    **Important :** Cette commande doit être exécutée sur votre **machine hôte** (celle sur laquelle vous avez lancé Vagrant), et non à l'intérieur des VMs Kubernetes. Le fichier `/etc/hosts` de votre machine hôte est utilisé par votre navigateur pour résoudre les noms de domaine. En ajoutant cette entrée, vous indiquez à votre machine hôte que `odoo.traefik.local` doit pointer vers l'IP du LoadBalancer fournie par MetalLB, qui est accessible depuis votre machine hôte via le réseau Vagrant.

    ```bash
    # Remplacez <IP_METALLB> par l'IP que vous avez copiée
    echo "<IP_METALLB> odoo.traefik.local" | sudo tee -a /etc/hosts
    ```

3.  **Accédez à Odoo :**
    Ouvrez votre navigateur et allez sur `http://odoo.traefik.local`. Vous devriez voir la page de configuration d'Odoo !

### 5. Évolution : Sécurisation en HTTPS

1.  **Générez un certificat auto-signé :**
    ```bash
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout tls.key -out tls.crt -subj "/CN=odoo.traefik.local"
    ```

2.  **Créez un Secret Kubernetes avec ces fichiers :**
    ```bash
    kubectl create secret tls traefik-tls-secret --key tls.key --cert tls.crt
    ```

3.  **Mettez à jour votre `Gateway` :**
    Modifiez `/traefik-odoo/gateway.yaml` pour ajouter un listener HTTPS.

    ```yaml
    # /traefik-odoo/gateway.yaml (version HTTPS)
    apiVersion: gateway.networking.k8s.io/v1
    kind: Gateway
    metadata:
      name: traefik-gateway
spec:
      gatewayClassName: traefik-gateway-class
      listeners:
      - name: http
        protocol: HTTP
        port: 80
        allowedRoutes:
          namespaces:
            from: Same
      - name: https
        protocol: HTTPS
        port: 443
        allowedRoutes:
          namespaces:
            from: Same
        tls:
          mode: Terminate
          certificateRefs:
          - name: traefik-tls-secret
    ```

4.  **Appliquez la mise à jour :**
    ```bash
    kubectl apply -f /traefik-odoo/gateway.yaml
    ```

Accédez maintenant à `https://odoo.traefik.local`. Votre navigateur affichera un avertissement de sécurité (car le certificat n'est pas signé par une autorité de confiance), mais vous pouvez l'accepter et voir qu'Odoo est servi en HTTPS.

---

## Cas B : Implémentation avec Nginx Gateway Fabric

Nginx est le standard de l'industrie pour les reverse-proxies. Leur implémentation de la Gateway API est nommée "Nginx Gateway Fabric".

*(Avant de commencer, vous pouvez nettoyer l'installation de Traefik pour éviter les conflits : `helm uninstall traefik` et `kubectl delete -f /traefik-odoo/`)*

### 1. Installation de Nginx Gateway Fabric
Nous utilisons également Helm.

```bash
# Ajouter le dépôt Helm de Nginx
helm repo add nginx-gateway https://oteemo.github.io/nginx-gateway-fabric-charts/

# Mettre à jour le dépôt
helm repo update

# Installer Nginx Gateway Fabric
helm install nginx-gateway-fabric nginx-gateway/nginx-gateway-fabric
```

### 2. Déploiement d'Odoo et PostgreSQL
**Bonne nouvelle !** Les manifestes de l'application (`postgres-*.yaml`, `odoo-*.yaml`) sont **exactement les mêmes**. C'est l'un des grands avantages de la Gateway API : la configuration de l'application est décorrélée de l'infrastructure réseau.

Vous pouvez réutiliser les mêmes fichiers que pour Traefik. Nous les plaçons dans un dossier virtuel `/nginx-odoo` pour la clarté.

```bash
# Si vous avez nettoyé le cluster, réappliquez les manifestes de l'application
kubectl apply -f /traefik-odoo/postgres-secret.yaml # ou le chemin vers votre copie
# ... et ainsi de suite pour tous les fichiers de l'application.
```

### 3. Configuration de la Gateway API (Nginx)

Ici, les fichiers changent car nous ciblons un autre contrôleur.

```yaml
# /nginx-odoo/gateway-class.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx-gateway-class
spec:
  controllerName: nginx.org/gateway-controller
```

```yaml
# /nginx-odoo/gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
spec:
  gatewayClassName: nginx-gateway-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Same
```

```yaml
# /nginx-odoo/http-route.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: odoo-http-route
spec:
  parentRefs:
  - name: nginx-gateway # On pointe vers la gateway Nginx
  hostnames: ["odoo.nginx.local"] # On utilise un autre nom d'hôte pour éviter les conflits
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: odoo-service # Le service applicatif reste le même
      port: 8069
```

**Appliquez ces configurations :**
```bash
kubectl apply -f /nginx-odoo/gateway-class.yaml
kubectl apply -f /nginx-odoo/gateway.yaml
kubectl apply -f /nginx-odoo/http-route.yaml
```

### 4. Accès en HTTP

1.  **Récupérez l'IP du LoadBalancer :**
    Le service de Nginx s'appelle `nginx-gateway-fabric`.

    ```bash
    kubectl get svc nginx-gateway-fabric -n nginx-gateway
    ```
    Copiez l' `EXTERNAL-IP`. Ce sera la même que pour Traefik si vous l'avez désinstallé, ou la suivante dans le pool MetalLB.

2.  **Modifiez votre fichier `hosts` :**
    **Important :** Comme pour le cas Traefik, cette commande doit être exécutée sur votre **machine hôte** (celle sur laquelle vous avez lancé Vagrant), et non à l'intérieur des VMs Kubernetes.

    ```bash
    # Remplacez <IP_METALLB> par la nouvelle IP
    echo "<IP_METALLB> odoo.nginx.local" | sudo tee -a /etc/hosts
    ```

3.  **Accédez à Odoo :**
    Ouvrez `http://odoo.nginx.local` dans votre navigateur.

### 5. Évolution : Sécurisation en HTTPS

Le processus est identique à celui de Traefik.

1.  **Générez un nouveau certificat pour le nouveau domaine :**
    ```bash
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout tls.key -out tls.crt -subj "/CN=odoo.nginx.local"
    ```

2.  **Créez le Secret (avec un nom différent pour éviter les conflits) :**
    ```bash
    kubectl create secret tls nginx-tls-secret --key tls.key --cert tls.crt
    ```

3.  **Mettez à jour la `Gateway` Nginx :**
    ```yaml
    # /nginx-odoo/gateway.yaml (version HTTPS)
    apiVersion: gateway.networking.k8s.io/v1
    kind: Gateway
    metadata:
      name: nginx-gateway
spec:
      gatewayClassName: nginx-gateway-class
      listeners:
      - name: http
        protocol: HTTP
        port: 80
        allowedRoutes:
          namespaces:
            from: Same
      - name: https
        protocol: HTTPS
        port: 443
        allowedRoutes:
          namespaces:
            from: Same
        tls:
          mode: Terminate
          certificateRefs:
          - name: nginx-tls-secret
    ```

4.  **Appliquez la mise à jour :**
    ```bash
    kubectl apply -f /nginx-odoo/gateway.yaml
    ```

Accédez à `https://odoo.nginx.local` pour vérifier que tout fonctionne.

## Conclusion

Félicitations ! Vous avez déployé une application complète via la Gateway API avec deux implémentations différentes.

**Qu'avons-nous observé ?**
- Les manifestes de l'application (Deployment, Service, etc.) sont **totalement indépendants** du choix du contrôleur de Gateway (Traefik ou Nginx).
- Les ressources `Gateway` et `HTTPRoute` sont **standardisées**. Leur structure est la même dans les deux cas.
- La seule vraie différence réside dans :
    1.  La commande d'installation du contrôleur (`helm install ...`).
    2.  Le nom du `controllerName` dans la `GatewayClass`.

Cet atelier démontre la puissance de la Gateway API : elle offre une abstraction standard qui permet de changer de technologie réseau sous-jacente sans jamais avoir à modifier les déploiements applicatifs. C'est une avancée majeure pour la portabilité et la maintenabilité des infrastructures sur Kubernetes.
