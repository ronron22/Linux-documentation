https://techoverflow.net/blog/2013/08/01/how-to-create-libsodium-nacl-deb-packages/

### Installer l'environnement de build
```
 apt-get install devscripts debhelper build-essential
```
et 
```
apt-get install dh-make-php php5-dev build-essential libmagic-dev debhelper
```
### Vérifier la version de l'API php 
```
php-config5 --phpapi
```
https://www.dotdeb.org/2008/09/25/how-to-package-php-extensions-by-yourself/

### j'exporte certaines variables
```
export DEBFULLNAME="gruik"
export DEBEMAIL="gruik@gruik"
```
### j'install la lib "manuellement"
```
pecl install libsodium
```
### je download les sources 
```
pecl donwload libsodium
```

### je génère les fichiers Debian
```
# https://bugs.launchpad.net/ubuntu/+source/dh-make-php/+bug/1039364
# !!!!! attention, bien appeller bash pour ne pas avoir d'erreur
bash dh-make-pecl --phpversion 5 --prefix php5-20100525- libsodium-1.0.2.tgz
```
### je génère les deb

```bash
cd php5-20100525-libsodium-1.0.2 &&  ./debian/rules binary
```

### points importants 

Standards-Version dans debian/control : ce n'est pas la version du paquet mais la version du standard Debian.
La version du paquet est incrémenté par la lecture du debian/changelog

### recompiler un paquet 

```bash
dpkg-buildpackage -r -B
```

