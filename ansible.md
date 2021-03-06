* [passer les variables d'ansible.cfg en variables d'environement](#passer-les-variables-d-ansible.cfg-en-variables-d-environement)
* [failed with error code 1 in /root/build/cryptography](#failed /root/build/cryptography)
* [ad-hoc MaJ de sécurite](#ad-hoc-MaJ-de-securite)
* [copier des données - local vers distant](#copier-des-données-local-vers-distant)
* [récupérer des fichiers distant](#récupérer-des-fichiers-distant)
* [installation des rôles de galaxy](#installation-des-rôles-de-galaxy)
* [unsupported parameter for module: gather_timeout] (#unsupported parameter for module: gather_timeout)


## passer les variables d ansible.cfg en variables d envirronement

```
export ANSIBLE_ROLES_PATH=$PWD/roles
export ANSIBLE_INVENTORY=$PWD/hosts-test
```

### ad-hoc MaJ de securite
`ansible messerveurs -s  -m apt -a "name=imagemagick,imagemagick-common state=latest"`

### copier des données local vers distant
```
ansible -i hosts  all  -m copy -a "src=roles/haproxy/files/haproxy.cfg dest=/etc/haproxy/"
```
### récupérer des fichiers distant

`ansible web -s -m fetch -a "src=/etc/apache2/apache2.conf  dest=/tmp/dd"`

### Mode Ad-hoc et sudo avec ansible 2
utiliser become 
```
ansible cache -m shell --become   -u ansible -a "apt-get -y install socat"
```

### Mode Ad-hoc et sudo avec ansible 1.*
```
ansible cache -m shell -s   -u ansible -a "apt-get -y install socat"
```

### copie de clés
```
ansible cache -s -m authorized_key -a "user=ansible key='ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDQuvjauHdo6MnfukMo8AO8uxcdeq9UudOrxXLLhsSu7oL9Hvk5NDSDcQvnhSXFQulqsHgB9PQ345I773cDanpZOkYn7WXa1tXkcchh3u7Dl5hg8KT8fXwkZLOeqIEHdXTm7CjoIY88lPRxo6fseHg7hqM3/ZJV+AfDfzoa2bY0VVOD5x9JWYioL69i6FAI9Pdsu9biHoBNrOMQXujjIrslP/fPygkR5/TOAcIQTnbjwNWb+bv9R2hghitmeUNNyMaehX+FDRboRPIyE+VWwMdCPXyF1MVA7A2CgN3oEoTB1X75PUR7NJses0dyvu8ZaJ7ZYhQLFydOyme9E/ bob@eollect'"
```
## Ansible et fact

http://jpmens.net/2012/07/15/ansible-it-s-a-fact/

Sur la cible, créer l'arbo
 
`/etc/ansible/facts.d` puis poser un script se terminant par `fact` exemple (du site de jpmens)

```
cat toto.fact
        #!/bin/sh
         
        COUNT=`who | wc -l`
        cat <<EOF
        {
           "ansible_facts" : {
              "users_logged_in" : $COUNT
           }
        }
        EOF
```
#### Test 1

Vérifier avec le module setup :

`ansible tagada7 -m setup -a "filter=ansible_local"`


#### Test2

Dans son playbook, il faut un
`gather_facts: yes`

Et dans la tâche
```
- name: pouet
debug: msg={{ansible_local.toto.ansible_facts.users_logged_in}}
```

failed /root/build/cryptography
-------------------------------

`apt-get install libffi-dev` 


### exclure certains serveur du traitement
`ansible-playbook --limit '!hoost1:!host2' yourPlaybook.yml`


# bonnes pratiques AWH 

- templates : créer des répertoires par 

## installation des rôles de galaxy

ajout d'un repo :
```
- src: https://github.com/geerlingguy/ansible-role-solr/archive/3.4.1.tar.gz
  version: master
  name: solr
```
installation du rôle
```
ansible-galaxy -r requirements.yml install -p roles/
```

### unsupported parameter for module: gather_timeout

Vous avez certainement deux versions différentes d'ansibles, une installée avec Debian l'autre avec PIP  

### debugging step by step

http://docs.ansible.com/ansible/latest/playbooks_startnstep.html

par exemple, je veux savoir quelle tâche créé un répertoire ?

Il faut ouvrir un **watch ls**  sur le serveur cible puis exécuter le playbook avec l'option **--step** ex **ansible-playbook playbook/monplay.yml --step** 
