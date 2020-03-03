![alt text](https://cdn.qwiklabs.com/ykt8NTzPWC6%2BW8mAljshFqjsAPrZ8bElG7SLw4kSrtU%3D "google cloud") 
# gcp_essentials 

ma première initiation au cloud computing 


### répertorier les noms des comptes actifs
```gcloud
gcloud auth list
```
### répertorier les ID de projet
```gcloud
gcloud config list project
```
### Installez NGINX
apt-get install nginx -y

### Vérifiez que NGINX est en cours d'exécution
ps auwx | grep nginx

### Dans Cloud Shell, créer une instance de machine virtuelle à partir gcloud 
gcloud compute instances create gcelab2 --machine-type n1-standard-2 --zone [your_zone]
###  définir la région et les zones par défaut utilisées par gcloud si vous travaillez toujours dans une même région/zone
```gcloud
gcloud config set compute/zone [your_zone]
gcloud config set compute/region [your_zone]
```
### Pour se connecter en SSH à votre instance 
```gcloud
gcloud compute ssh gcelab2 --zone [YOUR_ZONE]
```

## Créer une instance Windows Server dans Google Compute Engine, puis y accéder via le protocole RDP.

1. Dans la console GCP, accédez à Compute Engine > VM instances (Instances de VM), puis cliquez sur Create (Créer).
2. Dans la section Boot disk (Disque de démarrage), cliquez sur Change (Modifier) pour commencer à configurer le disque de démarrage.
Cliquez sur Windows Server 2012 R2 Datacenter, puis sur Select (Sélectionner). Conservez les valeurs par défaut de tous les autres paramètres.
3. Cliquez sur le bouton Create (Créer) pour créer l'instance.

Pour vérifier si le serveur peut accepter une connexion RDP, exécutez la commande suivante dans cloud shell
```gcloud
gcloud compute instances get-serial-port-output instance-1 --zone us-central1-a
```
NB: Répétez la commande jusqu'à ce que le résultat ci-dessous s'affiche
Finished running startup scripts.


### Se connecter à votre instance
1. Cliquez sur le nom de votre machine virtuelle.
2. Dans la section Remote Access (Accès à distance), cliquez sur le bouton Set Windows Password (Définir un mot de passe Windows)
3. Un nom d'utilisateur est généré.
Cliquez sur Set (Définir) afin de générer un mot de passe pour cette instance Windows. Cette opération peut prendre plusieurs minutes.
Copiez le mot de passe, puis enregistrez-le pour pouvoir vous connecter à l'instance.
4. Vous pouvez vous connecter via le protocole RDP directement depuis le navigateur à l'aide de l'extension Chrome RDP for Google Cloud Platform.
5. Cliquez sur RDP pour vous connecter.Vous êtes invité à installer l'extension RDP. Une fois celle-ci installée, GCP ouvre une page de connexion où vous pouvez saisir votre nom d'utilisateur et votre mot de passe Windows pour vous connecter. Collez le mot de passe que vous avez enregistré précédemment.

# Configurer des équilibreurs de charge réseau et HTTP
1. Définir la zone et la région par défaut de toutes les ressources
2. Créer plusieurs instances de serveur Web
...Pour créer les clusters de serveurs Web Nginx, créez les éléments suivants :
* Un script de démarrage, qui permettra à chaque instance de machine virtuelle de configurer le serveur Nginx au démarrage
* Un modèle d'instance, qui va utiliser le script de démarrage
* Un pool cible
* Un groupe d'instances géré, défini à partir du modèle d'instance

### le script de demarage et de configuration des serveurs nginx
```bash
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF
```
Créez un modèle d'instance, qui utilise le script de démarrage :
```gcloud
gcloud compute instance-templates create nginx-template \
         --metadata-from-file startup-script=startup.sh
 ```        
Celui-ci permet de disposer d'un point d'accès unique pour l'ensemble des instances d'un groupe
procéder ensuite à l'équilibrage de charge.
gcloud compute target-pools create nginx-pool

Créez un groupe d'instances géré, défini à partir du modèle d'instance :
```gcloud
gcloud compute instance-groups managed create nginx-group \
         --base-instance-name nginx \
         --size 2 \
         --template nginx-template \
         --target-pool nginx-pool
```
Listez les instances Compute Engine:
```gcloud
gcloud compute instances list
```
Configurons à présent le pare-feu, de sorte que vous puissiez vous connecter aux machines sur le port 80, via les adresses EXTERNAL_IP :
```gcloud
gcloud compute firewall-rules create www-firewall --allow tcp:80
```


Vous devriez pouvoir vous connecter à chacune des instances via son adresse IP externe, en saisissant le résultat de la commande précédente, soit http://EXTERNAL_IP/.


# Créer un équilibreur de charge réseau
* Créez un équilibreur de charge en réseau L3, qui va cibler votre groupe d'instances :
```gcloud
gcloud compute forwarding-rules create nginx-lb \
         --region us-central1 \
         --ports=80 \
         --target-pool nginx-pool
```
* Dressez la liste de toutes les règles de transfert Google Compute Engine de votre projet.
```gcloud
gcloud compute forwarding-rules list
```

Vous pouvez à présent accéder à l'équilibreur de charge en saisissant dans votre navigateur http://IP_ADDRESS/, où IP_ADDRESS désigne l'adresse renvoyée par la commande précédente.

# Créer un équilibreur de charge HTTP(S)

* Commençons par créer une vérification de l'état. Celle-ci consiste à vérifier que l'instance répond bien au trafic HTTP ou HTTPS :
```gcloud
gcloud compute http-health-checks create http-basic-check
```
* Définissons un service HTTP et mappons un nom de port sur le port correspondant au groupe d'instances. Le service d'équilibrage de charge peut désormais transférer le trafic vers le port nommé :

```gcloud
gcloud compute instance-groups managed \
       set-named-ports nginx-group \
       --named-ports http:80
```

* Créons un service de backend :
```gcloud
gcloud compute backend-services create nginx-backend \
      --protocol HTTP --http-health-checks http-basic-check --global
```
* Ajoutons le groupe d'instances au service de backend :
```gcloud
gcloud compute backend-services add-backend nginx-backend \
    --instance-group nginx-group \
    --instance-group-zone us-central1-a \
    --global
```
* Créez un mappage d'URL par défaut qui redirige toutes les requêtes entrantes vers vos instances :
```gcloud
gcloud compute url-maps create web-map \
    --default-service nginx-backend
```
* Créons un serveur proxy HTTP cible, qui va rediriger les requêtes vers votre mappage d'URL :
```gcloud
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map
```
* Créons une règle de transfert globale, qui va traiter et rediriger les requêtes entrantes. Une règle de transfert envoie le trafic vers un serveur proxy HTTP ou HTTPS spécifique, en fonction de l'adresse IP, du protocole IP et du numéro de port spécifié. À noter qu'une règle de transfert globale ne permet pas de spécifier plusieurs numéros de port.

```gcloud
gcloud compute forwarding-rules create http-content-rule \
        --global \
        --target-http-proxy http-lb-proxy \
        --ports 80
```
* La diffusion de votre configuration peut nécessiter plusieurs minutes, une fois la règle de transfert globale créée.
```gcloud
gcloud compute forwarding-rules list
```

# Orchestration de clusters avec Kubernetes Engine
* Démarrez une nouvelle session dans Cloud Shell et configurer la zone par defaut .
```gcloud
gcloud config set compute/zone us-central1-a
```
* Créer un cluster Kubernetes Engine
```gcloud
gcloud container clusters create [NOM-CLUSTER]
```
NB: règle du nom de cluster :Ce nom doit commencer par une lettre, se terminer par un caractère alphanumérique et comporter 40 caractères au maximum.

* Pour authentifier le cluster, exécutez la commande suivante en prenant soin de remplacer [NOM-CLUSTER] par le nom de votre cluster :
```gcloud
gcloud container clusters get-credentials [NOM-CLUSTER]
```
### Kubernetes Engine crée et gère les ressources de votre cluster à l'aide d'objets Kubernetes. Kubernetes fournit l'objet Déploiement pour déployer des applications sans état comme les serveurs Web. Les objets Service définissent des règles et l'équilibrage de charge pour accéder à votre application depuis Internet.
* Exécutez la commande kubectl create suivante dans Cloud Shell pour créer un objet Déploiement hello-server dans l'image du conteneur hello-app :
```gcloud
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
```
Cette commande Kubernetes permet de créer un objet Déploiement qui représente hello-server. Dans cette commande: --image désigne une image de conteneur à déployer. Dans ce cas, la commande extrait l'image d'exemple à partir d'un bucket Google Container Registry. gcr.io/google-samples/hello-app:1.0 indique la version de l'image à extraire. Si aucune version n'est spécifiée, la plus récente est utilisée.
* Créez maintenant un service Kubernetes, à savoir une ressource Kubernetes qui vous permet d'exposer votre application au trafic externe. Pour cela, exécutez la commande kubectl expose suivante :
```gcloud
kubectl expose deployment hello-server --type=LoadBalancer --port 8080
```
* Inspectez le service hello-server en exécutant la commande kubectl get :
```gcloud
kubectl get service
```
* Effectuer un nettoyage
Exécutez la commande suivante pour supprimer le cluster :
```gcloud
gcloud container clusters delete [NOM-CLUSTER]
```
