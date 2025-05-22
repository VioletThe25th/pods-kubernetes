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

### Exposer le pod

Afin d'exposer le pod, nous devons lancer cette commande : 
```bash
kubectl expose pod nginx-pod --port=80
```

Cependant, en l'état actuel des choses, la commande ne peut pas fonctionner. En effet, il manque le label dans le fichier de configuration `yaml`. Nous allons donc ajouter la partie suivante à la partie `metadata`: 
```yaml
metadata:
  name: nginx-pod
  labels:
    app: nginx
```

Suite à ces modifications, nous avons ce message de confirmation : `service/nginx-pod exposed`