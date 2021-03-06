# Strongswan for routing all trafic transparently for road warrior clients

ou **Router l'ensemble du trafic dans un tunnel ipsec strongswan en mode client mobile.**

auteur : ronron  
contact : '.@.\.'

## blabla

### pourquoi ce tuto et pourquoi un vpn

1. si tu recherche l'anonymat pour quelque raison que ce soit ;
2. si ta connexion Internet n'est pas de confiance ;
3. ben le reste..

> Il est simpliste (donc réducteur) d'associer l'anonymat sur Internet aux pervers et terroristes de tous poils, il s'agit là d'une grossière erreur, pourquoi accepterai-t-on d' être traqué par les sites web ? accepte-t-on nous d'être suivi ou surveillé dans la rue lorsque nous faisons des achats ? 

> La balance confort/liberté ne doit pas trop pencher sur le premier plateau. 

#### Avantage du vpn sur le proxy socks ou httpeu :

1. l'ensemble des flux sont routés ;
2. pas de réglages et de difficultés de prise en charge du proxy (fonction man in the middle toujours limite) ;
3. accès sécurisé obligatoire (secret partagé ou certificat), non ouvert par défaut.


### terminologie

> **road warrior** : normalement un cadre en déplacement, le plus souvent un ado pervers et boutonneux mattant des poneys en rut à longueur de soirée  
> **vpn** : outil pour matter des poneys en rut en loucedé  
> **ipsec** : système compliqué mais facile à prononcer  
> **strongswan** : un Cygne fort ?  
> **by-pass** : truc pour maigrir    
> **Internet** : endroit pleins de criminels et de **faique niouz**  
> **pandoc et markdown** : transformateur et format de document  
> **nat** : truc dans les cheveux des filles ~~ou des fiottes~~  
> **phases ipsec** : éléments du tunnel (phase 1, le tunnel de connexion - phase 2, **Le** ou **LES** tunnel(s) de transport)  

### rédaction et génération de ce document

et toc.. (oh oh oh)

```bash
pandoc -f markdown -t html5 --toc -s -o ~/ipsec.html ipsec-rw.md 
```

Document *css stylesheet less*.. 

### ingrédients

* un serveur *Linux* qui servira de passerelle *ipsec* (*apt install strongswan*) ; 
* un client ipsec sous Linux (ba encore *strongswan*..) ;
* de la patience.

## commençont

Il s'agit d'un *split tunneling* avec ip virtuel.

Nous utiliserons ikev2 (pourquoi pas ?)

Ressources :

* https://wiki.strongswan.org/projects/strongswan/wiki/ForwardingAndSplitTunneling
* https://wiki.strongswan.org/projects/strongswan/wiki/RouteBasedVPN
* https://wiki.strongswan.org/issues/2355

Vérifiez déjà que l'ip de sortie de votre VPN soit bien dans le ou la pays ou région désiré(e)  

* https://www.hostip.fr

### installons les programmes

compliqué..

```bash
~# apt install -y strongswan
```

sur la machine cliente et la passerelle.

#### quelques bases

```bash
~# ipsec start 
~# ipsec stop
~# ipsec reload 
~# ipsec restart
~# ipsec status
~# ipsec statusall
~# ipsec up "nom du tunnel"  # pour ceux qui sont en *auto=add*
```

### regardons les porcs..

Vérifier que les ports *ipsec* soient bien accessibles côté serveur.

* udp/4500
* udp/500

Si le status est *open/filtered*, c'est bon, c'est normal, c'est udp ^^ (voir la doc de nmap).

#### exemple

```bash
~# nmap -P0 -sU -p 500,4500   212.4.2.4

Starting Nmap 6.47 ( http://nmap.org ) at 2016-12-11 12:28 CET
Host is up (1.3s latency).
PORT     STATE         SERVICE
500/udp  open          isakmp
4500/udp open|filtered nat-t-ike
```

## config gateway (passerelle)

```bash
~# cat /etc/ipsec.conf
config setup
  # le debug
  charondebug="ike 1, knl 1, cfg 0"
  # limite le nombre de connexion je crois..
  uniqueids=no

conn ikev2-vpn
  # authentification par secret partagé
  authby=secret
  # démmarage auto
  auto=start
  # je désactive car les autres le font (beee..)
  compress=no
  # parcequ'on veut un tunnel.. 
  type=tunnel
  keyexchange=ikev2
  # en cas de problème de mtu
  fragmentation=yes
  # la flemme de regarder la doc
  forceencaps=yes
  # les algo..toussa
  ike=aes256-sha1-modp1024,3des-sha1-modp1024!
  esp=aes256-sha1,3des-sha1!
  # je crois que ce sont les période de scrutation du peer 
  dpdaction=clear
  dpddelay=300s
  # la flemme de regarder la doc
  rekey=no

  # notre patte *locale*
  left=163.172.21.190
  # ne pas spécifier les réseaux utilisables en phase 2, c'est autoriser l'accès vers tous.
  leftsubnet=%dynamic,0.0.0.0/0
  # gère les règles firewall de forward
  leftfirewall=yes
  # tente de pousser les serveur dns spécifiés ?
  rightdns=8.8.8.8,163.172.2.110
  # l'ip du client viendra de ce scope (ip virtuelle)
  rightsourceip=10.1.1.0/24
```

### le secret

```bash
~# cat /var/lib/strongswan/ipsec.secrets.inc
163.172.21.190 : PSK "cumulonimbus"
```

On ne spécifie pas de *right* car on attends des *road warriors* .

Attention à bien garder un espace entre l'adresse ip et le **:** 

### iptables première partie

Deux règles, une d'entrée, une de forward ont été automatiquement ajoutées (**ne pas le faire manuellement, le système s'en occupe**):

```bash
~# iptables-save
...
-A FORWARD -s 192.168.0.0/16 -i eth0 -m policy --dir in --pol ipsec --reqid 7 --proto esp -j ACCEPT
-A FORWARD -s 10.1.1.1/32 -i eth0 -m policy --dir in --pol ipsec --reqid 1 --proto esp -j ACCEPT
-A INPUT -s 192.168.0.0/16 -d 163.172.21.190/32 -i eth0 -m policy --dir in --pol ipsec --reqid 2 --proto esp -j ACCEPT
```

### Le routage

Pensez à activer le transfert entre interface 

```bash
sysctl net.ipv4.conf.all.forwarding=1
```

La table de routage ipsec est vide sur la gateway

```bash
~# ip route show table 220
~#
```

### iptables - seconde partie

Ajout du *nat* qui permettra la navigation des clients *vpn* (sinon pas d'internet avec une ip privée).

*Strongswan* ne *nat* pas de base, donc les paquets clients viendront de l'adresse définie dans le *rightsubnet* (192.168.0.0/16).

```bash
~# iptables -t nat -A POSTROUTING -s 10.1.0.0/16 -j MASQUERADE
```

### démarrage

Cette commande permet de rendre verbeux le démarrage du tunnel, n'oubliez pas qu'il est en démarrage automatique *auto=start*

```bash
ipsec up ikev2-vpn
```

## config client road warrior

```bash
~# cat /etc/ipsec.conf
config setup
  charondebug="ike 1, knl 1, cfg 0"
  uniqueids=no

conn ns7
  authby=secret
  auto=add
  compress=no
  type=tunnel
  keyexchange=ikev2
  fragmentation=yes
  forceencaps=yes
  ike=aes256-sha1-modp1024,3des-sha1-modp1024!
  esp=aes256-sha1,3des-sha1!
  dpdaction=clear
  dpddelay=300s
  rekey=no

  left=%defaultroute
  # à changer dans ipsec.secrets.inc
  leftsubnet=0.0.0.0/0

  right=163.172.21.190
  rightsubnet=0.0.0.0/0

conn local-net
  # les ips ci-dessous ne sont "traités" par le routage vpn classique 
  rightsubnet=163.172.21.190,212.129.2.222
  authby=never
  type=pass
  auto=route
```

### Qu'est-ce que la phase local-net ?

Cette phase sert à accéder aux ip normalement non-accessibles que sont les ip locales de la passerelle.  
Il s'agit d'un *by-pass*, les flux vers ces ips ne transitent pas dans le tunnel.

* https://wiki.strongswan.org/projects/strongswan/wiki/Bypass-lan
* https://serverfault.com/questions/709979/allow-strongswan-roadwarrior-to-access-local-lan

### le secret


```bash
~# cat /var/lib/strongswan/ipsec.secrets.inc 
192.168.11.5 163.172.21.190 : PSK  "cumulonimbus"
```

J'ai mis l'ip du *leftside*, il y a moyen de ne pas le faire, chercher la syntaxe peut-être un truc comme :

```bash
: 163.172.21.190 PSK  "cumulonimbus"
```

J'ai l'impression que même avec les ip virtuelles, le *rightside* est nécessaire.

### iptables 

On ajoute les règles permettant de router le trafic vers le tunnel.

Le faire manuellement en l'ajoutant à */etc/iptables/rules.v4* puis en rechargeant avec *service netfilter-persistent restart*

```bash
~# iptables -t nat -A POSTROUTING -s 10.1.0.0/16 -m policy --dir out --pol ipsec -j ACCEPT
~# iptables -t nat -A POSTROUTING -s 10.1.0.0/16 -j MASQUERADE
```

### le routage

Je pense que la seconde ligne correspond à la phase de *by-pass*

```bash
~# ip route show table 220
default via 192.168.11.1 dev wlx3476c5b50cd3 proto static src 10.1.1.1 
163.172.21.190 via 192.168.11.1 dev wlx3476c5b50cd3 proto static
...
```

Vérifions qu'une adresse donnée soit bien routée :

#### sans tunnel 

```bash
~# ip route get 8.8.8.8
8.8.8.8 via 192.168.11.1 dev wlx3476c5b50cd3 src 192.168.11.5 cache 
```

### ou by-passée

car présente dans le *rightsubnet* de tunnel *local-net*

```bash
~# ip route get 163.172.2.110
163.172.2.110 via 192.168.11.1 dev wlx3476c5b50cd3 table 220 src 192.168.11.5 cache 
```

#### avec tunnel 

```bash
~# ip route get 8.8.8.8
8.8.8.8 via 192.168.11.1 dev wlx3476c5b50cd3 table 220 src 10.1.1.1 cache 
```

remarquez la présence de la route dans la table ipsec (220)

### démarrage

Cette commande permet de rendre verbeux le démarrage du tunnel, n'oubliez pas qu'il est en démarrage manuel *auto=add*

```bash
ipsec up ns7
```

## status du tunnel

```bash
~# ipsec  statusall
Status of IKE charon daemon (strongSwan 5.5.1, Linux 4.9.0-8-amd64, x86_64):
  uptime: 5 seconds, since Oct 12 21:56:03 2018
  malloc: sbrk 2433024, mmap 0, used 425984, free 2007040
  worker threads: 11 of 16 idle, 5/0/0/0 working, job queue: 0/0/0/0, scheduled: 2
  loaded plugins: charon aes rc2 sha2 sha1 md5 random nonce x509 revocation constraints pubkey 
pkcs1 pkcs7 pkcs8 pkcs12 pgp dnskey sshkey pem openssl fips-prf gmp agent xcbc hmac gcm attr 
kernel-netlink resolve socket-default connmark stroke updown
Listening IP addresses:
  192.168.11.5
Connections:
         ns7:  %any...163.172.21.190  IKEv2, dpddelay=1s
         ns7:   local:  uses pre-shared key authentication
         ns7:   remote: [163.172.21.190] uses pre-shared key authentication
         ns7:   child:  dynamic === 0.0.0.0/0 TUNNEL, dpdaction=restart
   local-net:  %any...%any  IKEv1/2
   local-net:   local:  uses public key authentication
   local-net:   remote: uses public key authentication
   local-net:   child:  dynamic === 163.172.21.190/32 212.129.2.222/32 PASS
Shunted Connections:
   local-net:  dynamic === 163.172.21.190/32 212.129.2.222/32 PASS
Security Associations (1 up, 0 connecting):
         ns7[1]: ESTABLISHED 28 minutes ago, 192.168.11.5[192.168.11.5]...163.172.21.190[163.172.21.190]
         ns7[1]: IKEv2 SPIs: d8df57adec4c9268_i* 87ad9729dad1afd5_r, rekeying disabled
         ns7[1]: IKE proposal: AES_CBC_256/HMAC_SHA1_96/PRF_HMAC_SHA1/MODP_1024
         ns7[1]: Tasks active: IKE_MOBIKE 
         ns7{1}:  INSTALLED, TUNNEL, reqid 1, ESP in UDP SPIs: c42c9612_i ca13571d_o
         ns7{1}:  AES_CBC_256/HMAC_SHA1_96, 6117287 bytes_i (6680 pkts, 33s ago), 676695 bytes_o (6100 pkts, 0s ago), rekeying disabled
         ns7{1}:   10.1.1.1/32 === 0.0.0.0/0
```

### côté xfrm 

Vérifiez que les *policy* soient chargées

```bash
~# ip xfrm policy
src 0.0.0.0/0 dst 10.1.1.1/32 
	dir fwd priority 191808 ptype main 
	tmpl src 163.172.21.190 dst 192.168.11.5
		proto esp reqid 1 mode tunnel
src 0.0.0.0/0 dst 10.1.1.1/32 
	dir in priority 191808 ptype main 
	tmpl src 163.172.21.190 dst 192.168.11.5
		proto esp reqid 1 mode tunnel
src 10.1.1.1/32 dst 0.0.0.0/0 
	dir out priority 191808 ptype main 
	tmpl src 192.168.11.5 dst 163.172.21.190
		proto esp reqid 1 mode tunnel
src 163.172.2.110/32 dst 0.0.0.0/0 
	dir fwd priority 91808 ptype main 
src 163.172.2.110/32 dst 0.0.0.0/0 
	dir in priority 91808 ptype main
src 0.0.0.0/0 dst 163.172.2.110/32 
	dir fwd priority 91808 ptype main 
src 0.0.0.0/0 dst 163.172.2.110/32 
	dir out priority 91808 ptype main
```

### Les logs

Sur Debian, les évenements ipsec sont journalisés dans */var/log/daemon* 

## vérification

Toujours valider que le tunnel fasse bien sont job !!!

### avec curl

Faite un *curl ifconfig.io* pour valider que l'ip de sortie ai changée

#### Sans tunnel

```bash
~# curl ifconfig.io
180.14.50.49
```

#### Avec tunnel

```bash
~# curl ifconfig.io
163.172.21.190
```

### avec traceroute

#### sans tunnel 

```bash
~# traceroute  free.fr
traceroute to free.fr (212.27.48.10), 30 hops max, 60 byte packets
 1  gateway (192.168.11.1)  3.808 ms  3.795 ms  3.845 ms
 2  153.153.217.239 (153.153.217.239)  46.714 ms  47.123 ms  47.833 ms
 3  153.153.217.117 (153.153.217.117)  48.505 ms  48.583 ms  49.037 ms
 4  153.153.223.137 (153.153.223.137)  50.351 ms  51.562 ms  52.489 ms
 5  180.8.126.49 (180.8.126.49)  51.947 ms  54.388 ms  54.586 ms
 6  60.37.54.77 (60.37.54.77)  55.539 ms 122.1.245.57 (122.1.245.57)  54.694 ms 60.37.54.77 (60.37.54.77)  22.552 ms
 7  ae-6.r02.tokyjp05.jp.bb.gin.ntt.net (120.88.53.21)  36.131 ms ae-5.r02.tokyjp05.jp.bb.gin.ntt.net (120.88.53.17)  36.283 ms ae-6.r02.tokyjp05.jp.bb.gin.ntt.net (120.88.53.21)  36.095 ms
 8  ae-3.r31.tokyjp05.jp.bb.gin.ntt.net (129.250.3.29)  36.304 ms  36.380 ms ae-4.r31.tokyjp05.jp.bb.gin.ntt.net (129.250.3.57)  36.454 ms
 9  ae-4.r23.lsanca07.us.bb.gin.ntt.net (129.250.3.193)  144.452 ms ae-4.r23.snjsca04.us.bb.gin.ntt.net (129.250.5.78)  149.234 ms  149.934 ms
10  ae-41.r02.snjsca04.us.bb.gin.ntt.net (129.250.6.119)  149.053 ms  149.215 ms ae-2.r00.lsanca07.us.bb.gin.ntt.net (129.250.3.238)  149.428 ms
11  be3025.ccr41.lax04.atlas.cogentco.com (154.54.9.29)  149.364 ms ae-0.cogent.snjsca04.us.bb.gin.ntt.net (129.250.8.42)  149.078 ms  138.223 ms
12  be3360.ccr42.lax01.atlas.cogentco.com (154.54.25.149)  133.148 ms  133.624 ms  134.699 ms
13  be2932.ccr32.phx01.atlas.cogentco.com (154.54.45.161)  154.421 ms be3109.ccr21.slc01.atlas.cogentco.com (154.54.44.138)  158.994 ms be2932.ccr32.phx01.atlas.cogentco.com (154.54.45.161)  155.526 ms
14  be3037.ccr21.den01.atlas.cogentco.com (154.54.41.146)  192.061 ms be2930.ccr21.elp01.atlas.cogentco.com (154.54.42.78)  172.149 ms  172.254 ms
15  be2928.ccr42.iah01.atlas.cogentco.com (154.54.30.161)  181.886 ms be3036.ccr22.mci01.atlas.cogentco.com (154.54.31.90)  185.032 ms  187.006 ms
16  be2687.ccr41.atl01.atlas.cogentco.com (154.54.28.69)  198.453 ms be2831.ccr41.ord01.atlas.cogentco.com (154.54.42.166)  183.274 ms  184.225 ms
17  be2113.ccr42.dca01.atlas.cogentco.com (154.54.24.221)  213.470 ms be2112.ccr41.dca01.atlas.cogentco.com (154.54.7.157)  217.467 ms be2717.ccr21.cle04.atlas.cogentco.com (154.54.6.222)  193.232 ms
18  be2879.ccr22.alb02.atlas.cogentco.com (154.54.29.174)  217.204 ms  217.189 ms  217.256 ms
19  * * be3599.ccr31.bos01.atlas.cogentco.com (66.28.4.238)  213.176 ms
20  be2983.ccr42.lon13.atlas.cogentco.com (154.54.1.177)  264.150 ms be3518.agr21.par01.atlas.cogentco.com (130.117.50.42)  267.664 ms be2983.ccr42.lon13.atlas.cogentco.com (154.54.1.177)  264.875 ms
21  be12489.ccr42.par01.atlas.cogentco.com (154.54.57.70)  284.573 ms be12497.ccr41.par01.atlas.cogentco.com (154.54.56.130)  280.399 ms 149.14.152.218 (149.14.152.218)  280.850 ms
22  be3517.agr21.par01.atlas.cogentco.com (130.117.49.42)  299.161 ms * *
23  bzn-9k-4-be1004.intf.routers.proxad.net (78.254.249.2)  288.230 ms  283.075 ms  283.993 ms
24  bzn-9k-2-sys-be2000.intf.routers.proxad.net (194.149.161.242)  290.193 ms 194.149.166.61 (194.149.166.61)  281.349 ms bzn-9k-2-sys-be2000.intf.routers.proxad.net (194.149.161.242)  268.180 ms
25  bzn-9k-4-be1004.intf.routers.proxad.net (78.254.249.2)  275.950 ms  276.006 ms  276.455 ms
26  www.free.fr (212.27.48.10)  287.651 ms  289.589 ms bzn-9k-2-sys-be2000.intf.routers.proxad.net (194.149.161.242)  278.422 ms
```

#### Avec tunnel

```bash
~# traceroute  free.fr
traceroute to free.fr (212.27.48.10), 30 hops max, 60 byte packets
 1  gruik.com (163.172.21.190)  303.872 ms  303.882 ms  303.890 ms
 2  * 163-172-214-1.rev.poneytelecom.eu (163.172.21.1)  307.041 ms  307.449 ms
 3  195.154.2.128 (195.154.2.128)  310.245 ms  310.140 ms  310.194 ms
 4  51.158.8.26 (51.158.8.26)  310.511 ms  310.947 ms 51.158.8.24 (51.158.8.24)  311.259 ms
 5  195.154.2.103 (195.154.2.103)  320.219 ms 195.154.2.105 (195.154.2.105)  324.747 ms  323.479 ms
 6  195.154.3.208 (195.154.3.208)  326.246 ms bzn-crs16-1-be1500-t.intf.routers.proxad.net (212.27.58.49)  310.686 ms  305.905 ms
 7  194.149.166.61 (194.149.166.61)  307.016 ms 194.149.166.37 (194.149.166.37)  306.837 ms 194.149.166.61 (194.149.166.61)  310.639 ms
 8  bzn-9k-4-be1004.intf.routers.proxad.net (78.254.249.2)  309.878 ms  310.778 ms  310.596 ms
 9  bzn-9k-2-sys-be2000.intf.routers.proxad.net (194.149.161.242)  310.008 ms  310.828 ms  310.009 ms
10  bzn-9k-2.sys.routers.proxad.net (212.27.32.146)  310.081 ms  312.142 ms  313.356 ms
11  www.free.fr (212.27.48.10)  313.270 ms  312.277 ms  311.733 ms
```

#### avec tcpdump

Notre amie *tcpdump* nous permet de constater que le client arrive bien avec l'ip 10.1.1.1

```bash
~# timeout 5 tcpdump -vni any host linuxfr.org
tcpdump: listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
12:50:27.971895 IP (tos 0x0, ttl 64, id 48588, offset 0, flags [none], proto ICMP (1), length 84)
    10.1.1.1 > 88.191.250.176: ICMP echo request, id 11692, seq 6, length 64
```
