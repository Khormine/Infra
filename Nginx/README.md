# Installation de Nginx:

## Installation:

Installation de Nginx:
```shell
foo@bar:~$ apt update
foo@bar:~$ apt install nginx
```

Pour vérifier que nginx est activé:
```shell
foo@bar:~$ systemctl status nginx
* nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Fri 2023-03-10 09:57:09 UTC; 13s ago
```

## Configuration:

Pour ajouter un nom de domaine dans le serveur Nginx, il faut modifier le fichier `/etc/nginx/conf.d/default.conf`:

```shell
foo@bar:~$ nano /etc/nginx/conf.d/default.conf
server {
        listen   80;
        server_name  toto.fr;
        access_log  /srv/nginx/logs/toto.fr.log;

        location / {
                proxy_pass http://192.168.0.X;
        }
}
```

Pour vérifier qu'il n'y a pas d'erreurs dans le fichier de configuration:
```
foo@bar:~$ nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Pour que la configuration soit effective, il faut redémarrer le service:
```shell
foo@bar:~$ systemctl restart nginx 
```