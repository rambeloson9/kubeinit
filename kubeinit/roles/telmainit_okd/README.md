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





