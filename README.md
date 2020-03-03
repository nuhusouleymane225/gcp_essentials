![alt text](https://cdn.qwiklabs.com/ykt8NTzPWC6%2BW8mAljshFqjsAPrZ8bElG7SLw4kSrtU%3D "google cloud") 
# gcp_essentials 

ma première initiation au cloud computing 


### répertorier les noms des comptes actifs

gcloud auth list

### répertorier les ID de projet

gcloud config list project

### Installez NGINX
apt-get install nginx -y

### Vérifiez que NGINX est en cours d'exécution
ps auwx | grep nginx

### Dans Cloud Shell, créer une instance de machine virtuelle à partir gcloud 
gcloud compute instances create gcelab2 --machine-type n1-standard-2 --zone [your_zone]
###  définir la région et les zones par défaut utilisées par gcloud si vous travaillez toujours dans une même région/zone

gcloud config set compute/zone [your_zone]
gcloud config set compute/region [your_zone]
### Pour se connecter en SSH à votre instance 

gcloud compute ssh gcelab2 --zone [YOUR_ZONE]


## Créer une instance Windows Server dans Google Compute Engine, puis y accéder via le protocole RDP.

1. Dans la console GCP, accédez à Compute Engine > VM instances (Instances de VM), puis cliquez sur Create (Créer).
2. Dans la section Boot disk (Disque de démarrage), cliquez sur Change (Modifier) pour commencer à configurer le disque de démarrage.
Cliquez sur Windows Server 2012 R2 Datacenter, puis sur Select (Sélectionner). Conservez les valeurs par défaut de tous les autres paramètres.
3. Cliquez sur le bouton Create (Créer) pour créer l'instance.

Pour vérifier si le serveur peut accepter une connexion RDP, exécutez la commande suivante dans cloud shell

gcloud compute instances get-serial-port-output instance-1 --zone us-central1-a

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
gcloud compute instance-templates create nginx-template \
         --metadata-from-file startup-script=startup.sh
         
Celui-ci permet de disposer d'un point d'accès unique pour l'ensemble des instances d'un groupe
procéder ensuite à l'équilibrage de charge.
gcloud compute target-pools create nginx-pool

Créez un groupe d'instances géré, défini à partir du modèle d'instance :

gcloud compute instance-groups managed create nginx-group \
         --base-instance-name nginx \
         --size 2 \
         --template nginx-template \
         --target-pool nginx-pool
Listez les instances Compute Engine:
gcloud compute instances list

Configurons à présent le pare-feu, de sorte que vous puissiez vous connecter aux machines sur le port 80, via les adresses EXTERNAL_IP :
gcloud compute firewall-rules create www-firewall --allow tcp:80



Vous devriez pouvoir vous connecter à chacune des instances via son adresse IP externe, en saisissant le résultat de la commande précédente, soit http://EXTERNAL_IP/.
