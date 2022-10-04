****OKD 4.x******
****Bare metal*****


Installer un cluster provisionné par l'utilisateur sur système nu

Anvant d'installer OKD, il faut rendre sur l'url suivant

https://github.com/rambeloson9/kubeinit/tree/v2.1.0-branch

Cloner le repo

Ensuite, configurer le fichier inventory dans:

/kubeinit/roles/telmainit_okd/tests/inventory.ini

mettre votre adress IP pour faire l'installation

Executer le commande suivant

ansible-playbook -i /kubeinit/roles/telmainit_okd/tests/inventory.ini /kubeinit/roles/telmainit_okd/tests/main.yml

Le playbook lancera et installera tous les taches et les dependances inclus dans le role.

++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

Voici quelque script pour pour creer un cluster OKD sans utiliser ce playbook

Je testaits avec NGINX mais pas APACHE

Premierement, voici le quelque besoin logicielle,

Pour info, c'est une configuration minimum

* Bootstrap Node      ⇒ 4 CPU, 16 GB RAM, 100 GB Storage, Fedora CoreOS
* Control Plane Node  ⇒ 4 CPU, 16 GB RAM, 100 GB Storage, Fedora CoreOS
* Compute Node        ⇒ 2 CPU,  8 GB RAM, 100 GB Storage, Fedora CoreOS

* Bootstrap Node is needed only when bootstraping cluster.
 	
* Configure Manager Node, first.

Pré installation, configurer un DNS sur le Manager Node ou si vous n'avez pas encore une DNS pret à utiliser,

Ajouter les paramètres requis pour le cluster OKD à Dnsmasq

$ sudo vi /etc/dnsmasq.conf
--------------------------------------
# line 80 : add
# apps.(any cluster name).(domain name)/IP address
# [*.apps.okd4.srv.world] is resolved to [10.0.0.25]
address=/apps.okd4.srv.world/10.0.0.25

# line 145 : add domain name
domain=okd4.srv.world

--------------------------------------

$ sudo vi /etc/hosts

# [api], [api-int], [bootstrap] ⇒ fixed name
# [master-0] ⇒ hostname of each node you set
# [etcd-0], [_etcd-server-ssl._tcp] ⇒ CNAME of [master-0] and they are fixed name
# if adding more Control Planes : specify [etcd-(n)]
# ⇒ (IP address) (Hostname) etcd-1 _etcd-server-ssl._tcp
10.0.0.24   bootstrap
10.0.0.25   api api-int
10.0.0.40   master-0 etcd-0 _etcd-server-ssl._tcp
10.0.0.41   master-1 etcd-1 _etcd-server-ssl._tcp
10.0.0.42   master-2 etcd-2 _etcd-server-ssl._tcp

---------------------------------------------

$ sudo  systemctl restart dnsmasq

$ DNS=$(nmcli device show enp1s0 | grep ^IP4.DNS | awk '{print $2}')
$ nmcli connection modify enp1s0 ipv4.dns "10.0.0.25 $DNS"
$ nmcli connection modify enp1s0 ipv4.dns-search "okd4.srv.world"
$ nmcli connection up enp1s0

Ajoutez les paramètres requis pour le cluster OKD à Nginx.

$ sudo dnf -y install nginx-mod-stream

$ sudo vi /etc/nginx/nginx.conf

----------------------------------------------

server {
    	# lie 39 : change listening port
        listen       8080 default_server;
        listen       [::]:8080 default_server;

# add to the end
stream {
    upstream k8s-api {
        server 10.0.0.24:6443;
        server 10.0.0.40:6443;
        server 10.0.0.41:6443;
        server 10.0.0.42:6443;
    }
    upstream machine-config {
        server 10.0.0.24:22623;
        server 10.0.0.40:22623;
        server 10.0.0.41:22623;
        server 10.0.0.42:6443;
    }
    upstream ingress-http {
        server 10.0.0.40:80;
        server 10.0.0.41:80;
        server 10.0.0.42:6443;
    }
    upstream ingress-https {
        server 10.0.0.40:443;
        server 10.0.0.41:443;
        server 10.0.0.42:6443;
    }
    upstream ingress-health {
        server 10.0.0.40:1936;
        server 10.0.0.41:1936;
        server 10.0.0.42:6443;
    }
    server {
        listen 6443;
        proxy_pass k8s-api;
    }
    server {
        listen 22623;
        proxy_pass machine-config;
    }
    server {
        listen 80;
        proxy_pass ingress-http;
    }
    server {
        listen 443;
        proxy_pass ingress-https;
    }
    server {
        listen 1936;
        proxy_pass ingress-health;
    }
}


----------------------------------------------------------------

$ sudo  systemctl restart nginx

Si SELinux est activé, modifiez la policy.

$ setsebool -P httpd_can_network_connect on
$ setsebool -P httpd_graceful_shutdown on
$ setsebool -P httpd_can_network_relay on
$ setsebool -P nis_enabled on
$ semanage port -a -t http_port_t -p tcp 6443
$ semanage port -a -t http_port_t -p tcp 22623
$ semanage port -a -t http_port_t -p tcp 1936

Si Firewalld est en cours d'exécution, autorisez les ports de service

$ firewall-cmd --add-service={dns,http,https}
$ firewall-cmd --add-port={6443/tcp,22623/tcp,1936/tcp,8080/tcp}
$ firewall-cmd --runtime-to-permanent

Le preconfiguration est fini

+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

Telecharger le Pull Secret dans la site de RedHat

https://cloud.redhat.com/openshift/create/local

Creer un fichier ignition
Mais avant, voici les étapes à faire:

Telecharger Openshift client

Assurer si la dernierre version --------------> https://github.com/openshift/okd/releases/

$ wget https://github.com/openshift/okd/releases/download/4.10.0-0.okd-2022-03-07-131213/openshift-client-linux-4.10.0-0.okd-2022-03-07-131213.tar.gz \ https://github.com/openshift/okd/releases/download/4.10.0-0.okd-2022-03-07-131213/openshift-install-linux-4.10.0-0.okd-2022-03-07-131213.tar.gz


$ tar zxvf openshift-client-linux-4.10.0-0.okd-2022-03-07-131213.tar.gz
$ tar zxvf openshift-install-linux-4.10.0-0.okd-2022-03-07-131213.tar.gz
$ mv oc kubectl openshift-install /usr/local/bin/
$ chmod 755 /usr/local/bin/{oc,kubectl,openshift-install}
$ oc version

# générer une paire de clés SSH pour le nœud manager vers chaque nœud
# définissez la passphrase si vous en avez besoin ⇒ si elle est définie, elle a également besoin de l'agent SSH (définissez la phrase de passe sans mot de passe dans cet exemple)


$ ssh-keygen -q -N ""


exemple de path si vous connectez en tant que root

/root/.ssh/id_rsa

+++++++++++++++++++++++++++


Creation du fichier ignition

$ mkdir okd4
$ vi ./okd4/install-config.yaml

++++++++++++++++++++++++++

# [baseDomain] : specify base domain name
# [metadata.name] : specify any cluster name
# ⇒ (metadata.name).(baseDomain) is the same one with the name on DNSMasq you set like here
# [controlPlane.replicas] : specify number of Control Plane Nodes
# [pullSecret] : paste contents of Pull Secret you downloaded
# [sshKey] : paste contents of SSH key you generated above (public key)
apiVersion: v1
baseDomain: srv.world #ou okd.local
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: okd4
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{"auths":*****}'
sshKey: 'ssh-rsa *****'

+++++++++++++++++++++++++++++++++++++++


$ openshift-install create manifests --dir=okd4

$ openshift-install create ignition-configs --dir=okd4

$  cp ./okd4/{bootstrap.ign,master.ign,worker.ign} /usr/share/nginx/html/

$  chmod 644 /usr/share/nginx/html/{bootstrap.ign,master.ign,worker.ign}


++++++++++++++++++++++++++++++++++++++++++++++++++++++

Ensuite telecharger Fedora CoreOS ( le baremetal ISO) 

----> https://getfedora.org/en/coreos/download?tab=metal_virtualized&stream=stable&arch=x86_64

----> https://builds.coreos.fedoraproject.org/browser?stream=stable&arch=x86_64 (Older Version)

Définissez le fichier ISO sur le lecteur de DVD (physique ou virtuel) sur votre ordinateur sur lequel vous souhaitez installer Fedora CoreOS et allumez l'ordinateur.
Ensuite, Fedora CoreOS démarre à partir du DVD, appuyez sur la touche Entrée pour continuer, puis la connexion automatique s'exécute comme suit.

$ sudo coreos-installer install /dev/nvme0n1 --ignition-url=http://10.0.0.25:8080/bootstrap.ign --insecure-ignition --copy-network

$ sudo reboot
+++++++++++++++++++++++++++++++++++++++++++++++++++++


$ openshift-install --dir=okd4 wait-for bootstrap-complete

$ export KUBECONFIG=okd4/auth/kubeconfig
$ echo 'export KUBECONFIG=$HOME/okd4/auth/kubeconfig' >> ~/.bash_profile


++++++++++++++++++++++++++++++++++++++++++++++++++++++++++


$ oc get nodes






























