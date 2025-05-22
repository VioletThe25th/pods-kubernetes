# pods-kubernetes

N'ayant plus de cluster kubernetes d'installer de base sur mon ordinateur, j'ai passer par `minikube`, j'ai donc dû l'installer via `chocolatey` en mode admin sur `Powershell` :
```bash
choco install minikube
```
Puis lancer le cluster Kubernetes
```bash
minikube start
```
Nous voyons suite à la commande qu'un cluster Kubernetes a bien été créé sur `Lens`

---
Nous pouvons ensuite lancer la commande suivante pour se connecter au cluster
```bash
kubectl run nginx-pod --image=nginx:alpine --port=80
```
On voit ensuite la réponse : `pod/nginx-pod created`

Une fois ça de fait, il faut créer un fichier yaml de configuration du pod :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
```

Pour appliquer le code du `Yaml`, il faut lancer la commande suivante :
```bash
kubectl create -f pod-nginx.yaml
```
Cependant, il est important de `supprimer` le pod précédemment créé par la première commande `kubectl`.

## Exposer le pod

Afin d'exposer le pod, je dois lancer cette commande : 
```bash
kubectl expose pod nginx-pod --port=80
```

Cependant, en l'état actuel des choses, la commande ne peut pas fonctionner. En effet, il manque le label dans le fichier de configuration `yaml`. Je vais donc ajouter la partie suivante à la partie `metadata`: 
```yaml
metadata:
  name: nginx-pod
  labels:
    app: nginx
```

Suite à ces modifications, J'ai ce message de confirmation : `service/nginx-pod exposed`

---
## Créer un déploiement

J'ai tout d'abord créer un nouveau fichier `yaml` qui a pour objectif la création d'un namespace nommé `rolling` :
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: "rolling"
  labels:
    name: "rolling"
```

Afin de créer ce namespace, je lance la commande : 
```bash
kubectl apply -f rolling-namespace.yaml
```

Suite à cette commande, j'ai ce message de confirmation qui apparait : `namespace/rolling created`

Et en effet, le namespace est bien visible sur `Lens`.

---

Ensuite, il faut créer un `deployment`. Pour ceci, deux méthodes différentes sont fournis :

| Approche               | Avantages                                    | Inconvénients                          |
| ---------------------- | -------------------------------------------- | -------------------------------------- |
| **Impérative**         | Rapide pour tester ou faire des démos        | Peu traçable, pas de fichier versionné |
| **Déclarative (YAML)** | Versionnable, réutilisable, mieux pour CI/CD | Un peu plus de saisie initialement     |

### Méthode impérative

Se fait avec une ligne de commande : 
```bash
kubectl create deployment nginx-deployment -n rolling --image=nginx:1.14.2 --replicas=2 --port=80
```

### Méthode déclarative

Se fait avec un fichier `yaml` qui va créé un `Deployment` nommé `nginx-deployment` avec 2 pods `nginx:1.14.2`, dans le namespace `rolling`.

--- 
J'ai choisis de faire la méthode Déclarative. Voici donc le fichier `yaml` `nginx-deployment.yaml` : 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: rolling
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

Je lance le `yaml` avec la commande suivante : 
```bash
kubectl apply -f nginx-deployment.yaml
```

Je reçois le message de confirmation suivant : `deployment.apps/nginx-deployment created`

--- 
### Exposer le déploiement
Je crée un nouveau fichier `yaml` `nginx-service.yaml` :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-deployment
  namespace: rolling
  labels:
    app: nginx-deployment
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  selector:
    app: nginx
  type: NodePort  # pour permettre un accès via minikube
```
On peut voir que j'ai modifié le `selector` ainsi que le `type` dans les `specs` car j'ai utilisé `minikube`.

Afin d'appliquer les changements, je lance cette commande : 
```bash
kubectl apply -f nginx-service.yaml
```

Et j'ai bien le message de confirmation suivant : `service/nginx-deployment created`

Je peux vérifier que le service fonctionne en lançant la commande suivante : 
```bash
minikube service nginx-deployment -n rolling
```
Cette commande va permettre d'ouvrir mon navigateur sur l'URL `nginx`

Je peux également visualiser le service grâce à cette commande : 
```bash
kubectl describe service nginx-deployment -n rolling
```
Cette dernière affiche toutes les informations sur mon `service` :
```bash
Name:                     nginx-deployment
Namespace:                rolling
Labels:                   app=nginx-deployment
Annotations:              <none>
Selector:                 app=nginx
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.105.171.41
IPs:                      10.105.171.41
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31736/TCP
Endpoints:                10.244.0.6:80,10.244.0.7:80
Session Affinity:         None
External Traffic Policy:  Cluster
Internal Traffic Policy:  Cluster
Events:                   <none>
```
---
## Scale du déploiement

Afin de scaler un déploiement, on peut utiliser cette commande : 
```bash
kubectl scale deployment nginx-deployment -n rolling --replicas=<valeur>
```
Cela permet de modifier **le nombre souhaité de pods** dans le `déploiement`. Kubernetes ajuste automatiquement les pods, en en ajoutant ou en en suprimant certains.

Je vais pour ma part lancer cette commande : 
```bash
kubectl scale deployment nginx-deployment -n rolling --replicas=3
``` 
Ainsi :
- Kubernetes crée un 3ᵉ pod s’il n’y en avait que 2
- Et il place ce nouveau pod sur un noeud qui a assez de ressources disponibles (CPU, mémoire)

Si je fais la même commande en mettant `--replicas=4` :
- Kubernetes va essayer de programmer un 4ᵉ pod
- Si j'ai pllusieurs noeuds, alors il va essayer d'équilibrer sur un autre noeud
- Dans mon cas avec `minikube`, je n'ai qu'un seul noeud, **cela va alors lancer le 4ème pod sur le même noeud, s'il rèste assez de ressources**
- Si je n'ai pas assez de ressources, alors il restera en `pending`

---

## Utilisation des objets HPA
(HPA = Horizontal Pod Autoscaling)

L'objectif est maintenant de tracer les limites des pods grâce à la commande suivante : 
```bash
kubectl autoscale deployment/nginx-deployment -n rolling --min=5 --max=15 --cpu-percent=80
```
Cela va :
- limiter le nombre de pods entre **5** et **15**
- limiter la charge du CPU à 80% maximum

Suite à cette commande, j'ai le message de confirmation suivant qui apparait : `horizontalpodautoscaler.autoscaling/nginx-deployment autoscaled`

**Le HPA ne modifie pas directement le `ReplicaSet`, mais agit via le `Deployment`**.

Pour le vérifier, on peut lancer les commandes suivantes : 
```bash
kubectl get hpa --all-namespaces
```
On obtient le message suivant dans la console, attestant que les limites ont bien été tracés :
```bash
NAMESPACE   NAME               REFERENCE                     TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
rolling     nginx-deployment   Deployment/nginx-deployment   cpu: <unknown>/80%   5         15        5          2m
```
