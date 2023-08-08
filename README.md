
# Lago self-hosted


## Table of Contents

- [Requirements](#requirements)
- [Usage](#usage)
- [Configuration](#configuration)

## Requirements

1. Install Docker on your machine.

```
# Install Docker
sudo yum update -y
sudo yum install -y docker
sudo service docker start
sudo usermod -a -G docker ec2-user
# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

---

## Usage

### Set up environment configuration

```
echo "LAGO_RSA_PRIVATE_KEY=\"`openssl genrsa 2048 | base64`\"" >> .env
echo "LAGO_API_URL=api-domaine-name" >> .env
echo "LAGO_FRONT_URL=front-domaine-name" >> .env
echo "LAGO_PDF_URL=api-domaine-name" >> .env
```

### Docker swarm:

To run Lago using Docker Swarm mode, follow these steps:
+ initialize your Docker daemon as a swarm manager:

```
docker swarm init
```

or, join it to an existing swarm using

```
 docker swarm join
```

+ create an overlay network(used for internal communication betweeen the swarm services):

```
docker network create -d overlay public
```

:pushpin: Notice that we declare the 'public' network at the end of the lago docker-compose.yml:<br>

```
networks:
  public:
    external: true
```

:pushpin: Dont forget to connect the services in your docker compose file to the 'public' network, via :<br>

```
networks:
  - public
```
+ Deploy the stack to the Docker Swarm:

```
docker stack deploy -c docker-compose.yml lago
```
:pushpin:  to use env variables with swarm mode, source .env doesn't work, Notice this in our docker-compose file:

```
env_file: .env
```

## Configuration

# Environment variables
Lago uses the following environment variables to configure the components of the application. You can override them to customise your setup.
[check it out](https://doc.getlago.com/docs/guide/self-hosting/docker#environment-variables)

# Components
Lago uses the following containers:
| Container   | Role                                                                  |
| ----------- | ----------------------------------------------------------------------|
| front       | Front-end application                                                 |
| api         | API back-end application                                              |
| api_worker  | Asynchronous worker for the API application                           |
| api_clock   | Clock worker for the API application                                  |
| db          | Postgres database engine used to store application data               |
| redis       | Redis database engine used as a queuing system for asynchronous tasks |
| pdf         | PDF generation powered by Gotenberg                                   |


You can also use your own database or Redis server. To do so, remove the db and redis configurations from the docker-compose.yml file and update the environment variables accordingly.
