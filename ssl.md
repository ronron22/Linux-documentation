* [obtenir la date d'expiration d'un certificat, en ligne de commande] (#obtenir-la-date-d'expiration-d'un-certificat,-en-ligne-de-commande) 

https://www.sslshopper.com/article-most-common-openssl-commands.html

### obtenir la date d'expiration d'un certificat, en ligne de commande

`echo | openssl s_client -connect localhost:443 2>/dev/null | openssl x509 -noout -dates`

### obtenir le common name

openssl x509 -noout -subject -in cert.pem


### obtenir les Alternatives names

 openssl x509 -noout -text  -in /var/lib/dehydrated/certs/mugairyu.fr/cert.pem | grep DNS


