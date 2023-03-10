# Installation GitLab et GitLab Runner:

## Gitlab:

### Avec Docker Compose:

Fichier `docker-compose.yml`:
```yaml
services:
  web:
    image: 'gitlab/gitlab-ee:latest'
    container_name: robotenv
    restart: always
    hostname: 'pi04-gitlab.fr'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://192.168.0.X'
        # Add any other gitlab.rb configuration here, each on its own line
    ports:
      - '192.168.0.X:80:80'
      - '192.168.0.X:443:443'
      - '192.168.0.X:8022:22'
    volumes:
      - '$GITLAB_HOME/config:/etc/gitlab'
      - '$GITLAB_HOME/logs:/var/log/gitlab'
      - '$GITLAB_HOME/data:/var/opt/gitlab'
      - '/etc/hosts:/etc/hosts'
      - '/etc/apt/apt.conf.d/proxy.conf:/etc/apt/apt.conf.d/proxy.conf'
    shm_size: '256m'
```

Fichier `.env`:
```
GITLAB_HOME=/srv/gitlab
```

### Première connexion:

Visite l'URL du GitLab, et connecte toi avec le login `root` et le mot de passe obtenu avec la commande suivante:
```shell
foo@bar:~$ sudo docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

## GitLab runner:

### Installation du Docker Executor:

Fichier `docker-compose.yml`:
```yaml
version: '3.7'
services:
    gitlab-runner:
        container_name: gitlab-runner
        restart: always
        volumes:
            - '/var/run/docker.sock:/var/run/docker.sock'
            - '/data/gitlab-runner:/etc/gitlab-runner'
            - '/etc/hosts:/etc/hosts'
            - '/etc/apt/apt.conf.d/proxy.conf:/etc/apt/apt.conf.d/proxy.conf'
        image: 'gitlab/gitlab-runner:latest'
```

Pour enregistrer un runner, aller dans votre dépôt puis **Settings > CI/CD > Runners > Expand**. Vous trouverez l'URL et le token d'enregistrement pour le runner.

Installation du Docker Executor:
```shell
foo@bar:~$ mkdir -p /data/
foo@bar:~$ docker compose up -d
foo@bar:~$ docker exec -it gitlab-runner gitlab-runner register

Enter the GitLab instance URL (for example, https://gitlab.com/):
http://example.gitlab.com/
Enter the registration token:
token
Enter the descriptor for the runner:
[abcdefgh123]: runner1
Enter tags for the runner (comma-separated):
docker,debian
Enter an executor: docker, docker-ssh, parallels, virtualbox, docker-ssh+machine, custom, shell, ssh, docker+machine, kuberntes:
docker
Enter the default Docker image (for example, ruby:2.7):
debian:latest
```

### Image du runner pour Squash Autom:

Dockerfile de l'image de communication avec Squash Autom:
```Dockerfile
FROM python:3.8

RUN pip install --upgrade opentf-tools
```

Création de cette image avec le proxy de l'école:
```shell
foo@bar:~$ docker pull python:3.8
foo@bar:~$ docker build --no-cache --build-arg HTTP_PROXY='http://user:pwd@proxy-http.esisar.grenoble-inp.fr:3128' --build-arg HTTPS_PROXY='http://user:pwd@proxy-http.esisar.grenoble-inp.fr:3128' -t autom_runner .
```


### Configuration du Docker Executor:

Rajoutez dans /data/gitlab-runner/config.toml:
```shell
    pull_policy = "if-not-present"
    extra_hosts = ["pi04-squash-tm.fr:192.168.0.X"]
```