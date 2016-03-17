# ProjetFinalAdminSysteme
## Petit service LDAP composé de 4 machines virtuelles l'une agissant comme serveur, les trois autres comme clientes.
### Julien LELEU

--------

# Sommaire
- Le Serveur
    - Installation du serveur
    - Configuration du DNS
    - Hostname
    - Bind9
    - LDAP
    - Serveur NFS
- Le Client Debian
  - Installation et Configuration Montage NFS
  - Configuration du LDAP et NSS
  - Configuration du PAM
  - Configuration du fstab

# Serveur
## Installation du serveur
  - Dans un premier temps, munissez-vous du fichier mini-debian.iso disponible à l'adresse suivante : http://ftp.nl.debian.org/debian/dists/jessie/main/installer-amd64/current/images/netboot/

Lancez l'installation avec VMWare.

## Configuration lors de l'installation
  - Lors de l'installation, respectez les paramètres suivants:

### Paramètres de la machine

|Parametre|valeur|
|:----|:----|
|**Nom de la machine**|server|
|**Nom du domaine**|da2i.org|

### Miroir

|Parametre|valeur|
|:----|:----|
|**Pays**|France|
|**Package**|debian.polytech-lille.fr|
|**Proxy**|http://cache.univ-lille1.fr:3128|
|**Mot de passe super utilisateur**|root|

Installez automatiquement le disque dur(partitionnement automatique, fichiers multipliés avec repertoire /home séparé)

### Configuration DNS

Modifiez le fichier `/etc/network/interfaces` avec ces paramètres:

```
auto eth0
iface eth0  inet static
address 192.168.194.10
netmask 255.255.255.0
gateway 192.168.194.2
```
Pour relancer la configuration rebootez la machine ou relancez le service à l'aide de la commande suivante: `/etc/init.d/networking restart`

### Hostname

  - Lancez la commande `echo "server" > /etc/hostname` pour changer le nom de l'hote
  - Editez le fichier `/etc/hosts` et remplacez cette ligne `127.0.0.1 server.XXXXX server` par `192.168.194.10 server.da2i.org server`

  Une fois les modifications effectuées, redémarrez le service `networking`

## Bind9

Installez le paquet correspondant à **bind9** si celui-ci n'est pas déjà installé.
  - Editez le fichier `/etc/bind/named.conf.options`, insérez l'IP de VmPlayer et le DNS de l'université :

```
forwarders {
	192.168.194.2; 	#IP de vmPlayer
	172.18.48.31;	#DNS Universite
};
dnssec-validation no; #A modifier dans le fichier
```

  - Editez le fichier `/etc/bind/named.conf.local` :

```
zone "da2i.org" {
    type master;
    file "/etc/bind/zones/db.da2i.org";
};

zone "194.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192";
};
```

  - Créez le dossier `/etc/bind/zones`. Créez ensuite les fichiers `db.da2i.org` et `db.192` dans ce repertoire en y copiant respectivement le contenu de `/etc/bind/db.local` et `/etc/bind/db.127`.

  - Editez le fichier `/etc/bind/zones/db.da2i.org` avec ces informations:

```
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     server.da2i.org. webuser.da2i.org. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
da2i.org.       IN      NS      server.da2i.org.
da2i.org.       IN      A       192.168.194.10
;@      IN      NS      localhost.
;@      IN      A       127.0.0.1
;@      IN      AAAA    ::1
server  IN      A       192.168.194.10
gateway IN      A       192.168.194.2
debian  IN      A       192.168.194.20
archlinux       IN      A       192.168.194.30
freebsd IN      A       192.168.194.40
www     IN      CNAME   da2i.org.
```
  - Editez le fichier `/etc/bind/zones/db.192` avec ces informations:

```
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     server.da2i.org. webuser.da2i.org. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
;@      IN      NS      localhost.
;1.0.0  IN      PTR     localhost.
        IN      NS      server.
2       IN      PTR     gateway.da2i.org.
10      IN      PTR     server.da2i.org.
20      IN      PTR     debian.da2i.org.
30      IN      PTR     archlinux.da2i.org.
40      IN      PTR     freebsd.da2i.org.
```

  - Editez le fichier `/etc/resolv.conf` avec ces informations:

```
domain da2i.org
search da2i.org
nameserver 192.168.194.10
```

  - Ajoutez au fichier `/etc/network/interfaces` la ligne suivante: `dns-nameservers 192.168.194.10`.

Redémarrez ensuite le service bind9.

## Configuration de LDAP

  - Avant toute chose, installez les paquets `slapd ldap-utils`.
Pour installer et configurer facilement LDAP, suivez le tutoriel du site http://uid.free.fr/Ldap/ldap.html

|**Paramètres**|valeur|
|:-------------|:-----|
|Omit OpenLDAP server configuration ?|No|
|DNS domain name|da2i.org|
|Administrator password|root|
|Confirm password|root|
|Database backend to use|HDB|
|Do you want the database to be removed when slapd is purged?|No|
|Allow LDAPv2 protocol?|No|

### Création de l'arbre LDAP

  - Créez le fichier `ou.ldif` dans le dossier `/var/tmp` et insérez le contenu ci-dessous:

```
dn: ou=users,dc=da2i,dc=org
ou: users
objectClass: organizationalUnit

dn: ou=groupes,dc=da2i,dc=org
ou: groupes
objectClass: organizationalUnit
```

  - Effectuez la commande `invoke-rc.d slapd stop` pour eteindre le serveur suivi de `slapadd -c -v -l /var/tmp/ou.ldif` pour charger le fichier précédent.
### Création des utilisateurs

  - Pour cet exemple, nous appelerons cet utilisateur `robert`. Créez un fichier nommé `user1.ldif` dans le repertoire `/var/tmp` et y insérer le code suivant:

```
dn: cn=robert,ou=groupes,dc=da2i,dc=org
cn: robert
gidNumber: 1001
objectClass: top
objectClass: posixGroup

dn: uid=robert,ou=users,dc=da2i,dc=org
uid: robert
uidNumber: 1001
gidNumber: 1001
cn: robert
sn: robert
objectClass: top
objectClass: person
objectClass: posixAccount
objectClass: shadowAccount
loginShell: /bin/bash
homeDirectory: /home/robert
```

  - Ensuite, pour l'ajouter au serveur LDAP, effectuez la commande:

```
ldapadd -c -x -D cn=admin,dc=da2i,dc=org -W -f /var/tmp/user1.ldif
```

  - Définissez lui un mot de passe, avec la commande:

```
ldappasswd -x -D cn=admin,dc=da2i,dc=org -W -S uid=robert,ou=users,dc=da2i,dc=org
```

### Suppression d'utilisateurs

  - Si comme moi, il vous est arrivé malencontreusement de faire des erreurs lors de l'ajout d'un utilisateur, de faire une faute dans le nom ou encore d'oublier de changer le gid/uid pour chaque utilisateur, il existe une solution pour supprimer un utilisateur ainsi que son groupe.

  - Pour supprimer un utilisateur, effectuez la commande suivante:

```
ldapdelete -c -x -D cn=admin,dc=da2i,dc=org -W "uid=robert,ou=users,dc=da2i,dc=org"
```

- Pour supprimer son groupe, effectuez la commande suivante:

```
ldapdelete -c -x -D cn=admin,dc=da2i,dc=org -W "cn=robert,ou=groupes,dc=da2i, dc=org"
```

## Serveur NFS

  - Installez le paquet `nfs-kernel-server`.
  - Editez le fichier `/etc/exports` avec le code suivant:

```
/srv/serveur *.da2i.org(rw,sync,no_subtree_check)
```

  - (Re) Lancez le service `nfs-kernel-server`
  - Sur le serveur, créez le repertoire `/srv/home` qui contiendra les dossiers home de tous les utilisateurs LDAP
  - Pour chaque utilisateur, créez un repertoire `/srv/home/<NomUtilisateur>`
  - Sur le client, créez le repertoire `/home` et montez le à l'aide de la commande suivante:

```
  mount -t nfs 192.168.194.10:/srv/home/<NomUtilisateur> /home
```

# Client Debian

  - Pour créer et configurer votre machine debian, reportez vous au tutoriel d'installation de la partie serveur.

## Configuration du LDAP

  - Vérifiez qu'avec la commande `id <NomUtilisateur>`, la commande vous renvoie bien `Not such user`
  - Installez les paquets `libnss-ldap libpam-ldap nscd`
  - Renseignez les valeurs suivantes:

```
LDAP server URI: ldap://192.168.194.10/
Distinguished name of the search base: dc=da2i,dc=org
LDAP version to use: 3
Does the LDAP database require login? No
Special LDAP privileges for root? No
Make the configuration file readable/writable by its owner only? No
Allow LDAP admin account to behave like local root? Yes
Make local root Database admin. No
Does the LDAP database require login? No
LDAP administrative account: cn=admin,dc=da2i,dc=org
LDAP administrative password: root
DLocal crypt to use when changing passwords. md5
```

  - Si vous avez commis une erreur ou sauté une étape, vous pouvez reconfigurer les paquets avec `dpkg-reconfigure {paquet}`
  - Editez le fichier `/etc/libnss-ldap.conf` avec le contenu suivant:

```
base dc=da2i,dc=org
uri ldap://192.168.194.10/
```

  - Editez le fichier `/etc/nsswitch.conf` avec le contenu suivant pour activer le module LDAP NSS:

```
passwd:         files ldap
group:          files ldap
```

### Configuration du PAM

  - Dans l'étape qui suit, nous allons simplement modifier et/ou ajouter des valeurs à certains fichiers.
  - Editez le fichier `/etc/pam.d/common-account` avec les informations suivantes:

```
account [success=2 new_authtok_reqd=done default=ignore]        pam_unix.so
account [success=1 default=ignore]      pam_ldap.so

account requisite                       pam_deny.so

account required                        pam_permit.so
```

  - Editez le fichier `/etc/pam.d/common-auth` avec les informations suivantes:

```
auth    [success=2 default=ignore]      pam_unix.so nullok_secure
auth    [success=1 default=ignore]      pam_ldap.so use_first_pass

auth    requisite                       pam_deny.so

auth    required                        pam_permit.so
```

  - Editez le fichier `/etc/pam.d/common-passwd` avec les informations suivantes:

```
password        [success=2 default=ignore]      pam_unix.so obscure sha512
password        [success=1 user_unknown=ignore default=die]     pam_ldap.so use_authtok try_first_pass

password        requisite                       pam_deny.so

password        required                        pam_permit.so
```
  - Editez le fichier `/etc/pam.d/common-session` avec les informations suivantes:

```
session [default=1]                     pam_permit.so

session requisite                       pam_deny.so

session required                        pam_permit.so

session required        pam_unix.so
```

  - Pour prendre en compte ces modifications, rebootez votre client à l'aide de la commande `reboot`.

### Configuration de fstab

- Pour réussir à monter le disque `/srv/home` sur le client(qui permettra à chaque client d'avoir son `/home`), modifiez le fichier `/etc/fstab` avec ces valeurs:

```
192.168.194.10:/srv/serveur 	/home 	nfs4 	auto 	0 	0
```

- N'oubliez pas de supprimer la ligne suivante: `UUID=dd3a0110-1482-4592-8c95-3ba3ebfb8e84 /home           btrfs   defaults    $`
