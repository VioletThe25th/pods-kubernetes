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

--- 

## Classe de stockage

Je peux lister les classes de stockages disponibles grâce à cette commande : 
```bash
kubectl get storageclass
```

J'obtiens donc ce message dans la console :
```bash
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  67m
```

Je ne peux cependant pas noter de différences entre les `VOLUMEBINDINGMODE` étant donné que j'en ai qu'un qui est lancé actuellement. Mais de ce que j'ai pu trouver sur internet, et avec les informations présentes sur l'image d'exemple, il existe le mode `Immediate` comme j'ai moi, et le mode `WaitForFirstConsumer` :

| Mode                   | Comportement                                           |
| ---------------------- | ------------------------------------------------------ |
| `Immediate`            | Volume créé **immédiatement** à la demande             |
| `WaitForFirstConsumer` | Volume créé **lorsqu’un pod est planifié** sur un nœud |

## Création d'un PVC
(PersistentVolumeClaim)

Je crée un fichier `yaml` `pvc.yaml` qui contient la configuration suivante :
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    name: azure-managed-disk
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-csi
  resources:
    requests:
      storage: 5Gi
```

Le fichier va créer un **PersistentVolumeClaim** nommée `azure-managed-disk`.
Afin d'appliquer les changements : 
```bash
kubectl apply -f pvc.yaml
```
Je vois ce message de confirmation dans la console : `persistentvolumeclaim/azure-managed-disk created`

Je lance ensuite la commande `kubectl get pvc` et je vois ce message :
```bash
NAME                 STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
azure-managed-disk   Pending                                      managed-csi    <unset>                 74s
```

---

## Utiliser le volume et déployer un pod

Je crée le fichier `pod.yaml` :
```bash
kind: Pod
apiVersion: v1
metadata:
  name: mypod
spec:
  containers:
    - name: mypod
      image: mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 250m
          memory: 256Mi
      volumeMounts:
        - mountPath: "/mnt/azure"
          name: volume
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: azure-managed-disk
```

Ce dernier va déployer un pod `mypod` sur le volume `/mnt/azure` qui provient du PVC précédent (azure-managed-disk)

Afin de l'appliquer, je lance la commande suivante : 
```bash
kubectl apply -f pod.yaml
```

J'ai le message de confirmation suivant : `pod/mypod created`

Afin de vérifier qu'il est bien lancé, je lance la commande suivante : 
```bash
kubectl get pods
```

Je vois ce résultat dans la console : 
```bash
NAME        READY   STATUS    RESTARTS   AGE
mypod       0/1     Pending   0          20s
nginx-pod   1/1     Running   0          75m
```

---

### Créer un fichier depuis le pod

Afin que cette étape fonctionne, je dois passer par plusieurs étapes intermédiaires, dû au fait que j'utilise `minikube`. Comme je l'avais dis plus haut, `minikube` n'accepte qu'un pod à la fois. Pour ce faire, je dois utiliser une `StorageClass` compatible **hostPath**.
Je vais donc passer par le `storageClassName` : `standard` de `minikube`, au lieu de `managed-csi` dans `pvc.yaml`. Je vais relancer tout grâce à cette ligne de commande : 
```bash
kubectl delete pod mypod
kubectl delete pvc azure-managed-disk

kubectl apply -f pvc.yaml
kubectl apply -f pod.yaml
```


Je vais suivre les étapes suivantes : 
1. Ouvrir un shell dans le pod
2. Créer un fichier dans le volume
3. Quitter le pod (shell)
4. Supprimer le pod
5. Vérifier que le PVC et PV existent encore
6. Recréer le pod
7. Rouvrir un shell dans le nouveau pod
8. Vérifier que le fichier est toujours là

#### 1. Ouvrir un shell dans le pod
```bash
kubectl exec mypod -it -- sh
```

#### 2. Créer un fichier dans le volume
```bash
cd /mnt/azure
echo "Coucou" > myfile.txt
```

On va ensuite voir le fichier : 
```bash
cat myfile.txt
```

On voit le message suivant dans la console : 
```bash
/mnt/azure # cat myfile.txt
Coucou
```
#### 3. Quitter le pod
```bash
exit
```

#### 4. Supprimer le pod
```bash
kubectl delete -f pod.yaml
```

On voit ce message de confirmation : `pod "mypod" deleted`

#### 5. Vérifier que le PVC et PV existent encore
```bash
kubectl get pvc
kubectl get pv
```

On voit le message de confirmation suivant : 
```bash
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
azure-managed-disk   Bound    pvc-dafcf652-8150-4639-9e3a-7d65383165b9   5Gi        RWO            standard       <unset>                 6m19s
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-dafcf652-8150-4639-9e3a-7d65383165b9   5Gi        RWO            Delete           Bound    default/azure-managed-disk   standard       <unset>                          6m19s
```

#### 6. Recréer le pod
Avec la même commande que lors de sa création
```bash
kubectl apply -f pod.yaml
```

#### 7. Rouvrir un shell dans le nouveau pod
```bash
kubectl exec mypod -it -- sh
```

#### 8. Vérifier que le fichier est toujours là
```bash
cd /mnt/azure
ls
cat myfile.txt
```

On voit bien lors de l'execution de la commande `ls`, que le fichier `myfile.txt` est toujours là :
```bash
/ # cd /mnt/azure
/mnt/azure # ls
myfile.txt
```
Si je lance la commande 
```bash
cat myfile.txt
```
Et on reçoit bien le message `Coucou` 
```bash
/mnt/azure # cat myfile.txt
Coucou
```

---

## Variables et secrets

### Configmap
(Permet de passer des informations aux Pods comme des variables d'environnement)

J'ai créé le `nginx-configmap.yaml` comprenant le code `yaml` suivant : 
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  CUST_NGINX_PORT: "8080"
  CUST_NGINX_WORKER_PROCESSES: "2"
```

J'applique la configmap dans le clsuter grâce à la commande suivante : 
```bash
kubectl apply -f nginx-configmap.yaml
```

Je reçois bien le message de confirmation suivant : `configmap/nginx-config created`

Afin de vérifier que c'est bien créer je peix lancer la commande suivante : 
```bash
kubectl get configmap
```

Je vois le résultat suivant dans la console : 
```bash
NAME               DATA   AGE
kube-root-ca.crt   1      118m
nginx-config       2      82s
```

### Utilisation de la configmap

Utilisation d'un `envFrom` en ajoutant ceci à la fin du fichier `pod-nginx.yaml` : 
```yaml
envFrom:
        - configMapRef:
            name: nginx-config
```

Je lance ensuite la commande suivante : 
```bash
kubectl apply -f pod-nginx.yaml
```

Après ça, je me connecte au pod : 
```bash
kubectl exec nginx-pod -it -- sh
```

Puis je vérifie que les variables d'environnement sont bien présentes : 
```bash
env | grep CUST
```

J'obtiens ce message : 
```bash
/ # env | grep CUST
CUST_NGINX_PORT=8080
CUST_NGINX_WORKER_PROCESSES=2
```

### DockerHub

#### 1. Pull l'image et la renommée

```bash
docker pull nginx:alpine
docker tag nginx:alpine violetgrace/mynginx:alpine
```

#### 2. Push l'image sur DockerHub

Je dois d'abord me connecter :
```bash
docker login
```
Puis push l'image :
```bash
docker push violetgrace/mynginx:alpine
```

Je rend ensuite le repo privée sur `DockerHub`

Je lance esuite cette commande pour accéder au repo privée : 
```bash
kubectl create secret docker-registry regcred \
  --docker-server=docker.io
  --docker-username=violetgrace \
  --docker-password=<your-password> \
  --docker-email=bilgerjeremy5705@gmail.com \
  --namespace=default
```

je vois bien le message de confirmation : `e=default secret/regcred created`

Je vérfie ensuite en lançant la commande : 
```bash
kubectl get secrets
```

Et j'ai bien le message suivant dans la console : 
```bash
NAME            TYPE                             DATA   AGE
regcred         kubernetes.io/dockerconfigjson   1      24s
```


Je crée le yaml suivant : 
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg-container
    image: violetgrace/mynginx:alpine
  imagePullSecrets:
  - name: regcred
```

Je lance la commande suivante : 
```bash
kubectl apply -f pod-private.yaml
```

