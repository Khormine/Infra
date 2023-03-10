# Installation et configuration de Squash TM et Squash Autom:

## Squash TM:

### Installation avec Docker Compose:
Fichier `docker-compose.yml`:
```yaml
version: '3.7'
services:
  squash-tm_db:
    image: mariadb:10.2.22-bionic
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_DATABASE}
    volumes:
      - "/srv/infra/mysql:/var/lib/mysql"

  squash-tm:
    image: squashtest/squash-tm
    restart: always
    depends_on:
      - squash-tm_db
    environment:
      SQTM_DB_TYPE: mysql
      SQTM_DB_USERNAME: ${DB_USER}
      SQTM_DB_PASSWORD: ${DB_PASSWORD}
      SQTM_DB_NAME: ${DB_DATABASE}
    extra_hosts:
      - "pi04-gitlab.fr:192.168.0.3"
      - "pi04-squash-tm.fr:192.168.0.3"
      - "pi04-squash-autom.fr:192.168.0.3"
    ports:
      - 192.168.0.4:8090:8080/tcp
    links:
      - squash-tm_db:mysql
    volumes:
      - "/srv/infra/squash-tm/logs:/opt/squash-tm/logs"
      - "/srv/infra/squash-tm/plugins:/opt/squash-tm/plugins"
      - "/etc/hosts:/etc/hosts"
      - "/etc/apt/apt.conf.d/proxy.conf:/etc/apt/apt.conf.d/proxy.conf"
```

Ficher `.env`:
```
DB_USER=squashdbuser
DB_PASSWORD=pwd
DB_ROOT_PASSWORD=pwd
DB_DATABASE=squashtm
```


## Installtion Squash Autom:

### Création d'une clé publique:

Créer ses propres clés privé et publique qui seront utilisés par l'orchestrateur:
```
openssl genrsa -out trusted_key.pem 4096
openssl rsa -pubout -in trusted_key.pem -out trusted_key.pub
```

### Instalation avec Docker Compose:

Fichier `docker-compose.yml`:
```yaml
version: "3.7"
services:
  orchestrator:
    container_name: orchestrator
    image: squashtest/squash-orchestrator:latest
    restart: always
    ports:
    - "192.168.0.X:7774:7774"    # receptionist
    - "192.168.0.X:7775:7775"    # observer
    - "192.168.0.X:7776:7776"    # killswitch
    - "192.168.0.X:38368:38368"  # eventbus
    - "192.168.0.X:24368:24368"  # agent channel
    - "192.168.0.X:12312:12312"  # quality gate
    environment:
      SSH_CHANNEL_HOST: $SSH_CHANNEL_HOST
      SSH_CHANNEL_PORT: $SSH_CHANNEL_PORT
      SSH_CHANNEL_USER: $SSH_CHANNEL_USER
      SSH_CHANNEL_PASSWORD: $SSH_CHANNEL_PASSWORD
      SSH_CHANNEL_TAGS: ssh,linux,robotframework
    extra_hosts:
      - "pi04-gitlab.fr:192.168.0.X"
      - "pi04-squash-tm.fr:192.168.0.X"
      - "pi04-squash-autom.fr:192.168.0.X"
    volumes:
      - "$KEY:/etc/squashtf/trusted_key.pub"
      - "/etc/hosts:/etc/hosts"
      - "/etc/apt/apt.conf.d/proxy.conf:/etc/apt/apt.conf.d/proxy.conf"
```

Fichier `.env`:
```
SSH_CHANNEL_HOST=192.168.0.x
SSH_CHANNEL_PORT=8022
SSH_CHANNEL_USER=otf
SSH_CHANNEL_PASSWORD=secret
KEY=/root/data/trusted_key.pub
```

### Installation du Runner RobotFramework:

Fichier `docker-compose.yml`:
```yaml
version: '3.7'
services:
  robot-framework-runner:
    container_name: robotenv
    environment:
      PASSWORD_ACCESS: true
      USER_NAME: $USER_NAME
      USER_PASSWORD: $USER_PASSWORD 
    ports:
      - '192.168.0.6:8022:22'
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
    volumes:
      - '/etc/hosts:/etc/hosts'
      - '/etc/apt/apt.conf.d/proxy.conf:/etc/apt/apt.conf.d/proxy.conf'
    image: 'opentestfactory/robot-framework-runner:latest'
```

Fichier `.env`:
```
USER_NAME=otf
USER_PASSWORD=secret
```

### Configuration sur Gitlab:

Allez dans **Settings > CI/CD > Variables > Expand" et ajoutez les variables *OPENTF_CONFIG* et *OPENTF_TOKEN*:
```yaml
apiVersion: opentestfactory.org/v1alpha1
contexts:
- context:
    orchestrator: my_orchestrator
    user: me
  name: my_orchestrator
current-context: my_orchestrator
kind: CtlConfig
orchestrators:
- name: my_orchestrator
  orchestrator:
    insecure-skip-tls-verify: false
    services:
      receptionist:
        port: 7774
      agentchannel: 
        port: 24368
      eventbus:
        port: 38368
      killswitch:
        port: 7776
      observer:
        port: 7775
      qualitygate:
        port: 12312
    server: http://pi04-squash-tm.fr
users:
- name: me
  user:
    token: aa
```

Pour la variable *OPENTF_TOKEN*, il faut renseigner le token JWT généré par la commande suivante dans la machine où il y a Squash Autom:
```shell
opentf-ctl generate token using trusted_key.pem
```

### Création d'un workflow:

Dans votre dépôt, créez un dossier `.opentf/workflows` et ajoutez le fichier suivant:
```yaml
metadata:
  name: Robot Framework Example
jobs:
  keyword-driven:
    runs-on: robotframework
    steps:
    - uses: actions/checkout@v2
      with:
        repository: https://your-scm-manager/RobotDemo.git
    - uses: robotframework/robot@v1
      with:
        datasource: RobotDemo/keyword_driven.robot
```

Ce fichier donne les instructions que doit faire le runner de Squash Autom. Ici, il clone le dépôt `RobotDemo` et lance les tests dans `keyword_driven.robot`.

Pour plus d'information sur la syntaxe des workflows, voir  [ici](https://opentestfactory.org/specification/workflows.html).

### Intégration avec Gitlab CI:

Une fois le fichier de workflow créé dans le dépôt, allez dans **CI/CD > Editor** et définissez votre pipeline de test. Par exemple:
```yaml
default:
  image: autom_runner

stages:            # List of stages for jobs, and their order of execution
  - test

opentf-workflow:   # This job runs in the build stage, which runs first.
  stage: test
  tags: 
    - docker
  script:
    - RESULT=$(opentf-ctl run workflow .opentf/workflows/workflow.yaml)
    - echo $RESULT
    - WORKFLOW_ID=$(echo $RESULT | awk -F ' ' '{print $2}')
    - opentf-ctl get workflow $WORKFLOW_ID --watch
    - opentf-ctl get qualitygate $WORKFLOW_ID

```

La commande `opentf-ctl run` envoie l'information de faire les tests à l'orchestrateur de Squash Autom en lui envoyant le fichier de workflow.
La commande `opentf-ctl get workflow <ID> --watch` permet de receuillir l'avancée des tests sur le runner de tests.
La commande `opentf-ctl get qualitygate <ID>` permet de savoir si les tests sont passés ou non. Si la qualité est `strict` alors tous les tests sont passés, si la qualité est `passing` alors certains tests ont échoués.