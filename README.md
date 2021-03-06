# Docker-registry installation
Collection of docker files and instructions on how to setup and run a private docker registry (v2) behind nginx as a reverse proxy.
Nginx is configured with TLS and basic authentication.

## Running docker registry

### creation

Create a volume container for the docker registry:
```
sudo docker create -v /etc/registry -v /srv/registry-data --name registry_volume registry:2.0
```

Create a container running the registry:
```
sudo docker create -p 127.0.0.1:5000:5000 --volumes-from registry_volume --name registry registry:2.0 /etc/registry/config.yaml
```

### configuration

Use an ubuntu based container with the registry volumes mounted to create and 
edit the default configuration:
```
sudo docker run -ti --rm --volumes-from registry_volume ubuntu /bin/bash
```

Example config.yaml:
```
version: 0.1
log:
  level: debug
  fields:
    service: registry
    environment: development
storage:
  cache:
      layerinfo: inmemory
  filesystem:
      rootdirectory: /srv/registry-data/
http:
  addr: :5000
  secret: asecretforlocaldevelopment
  debug:
      addr: localhost:5001
redis:
  addr: localhost:6379
  pool:
    maxidle: 16
    maxactive: 64
    idletimeout: 300s
  dialtimeout: 10ms
  readtimeout: 10ms
  writetimeout: 10ms
notifications:
  endpoints:
      - name: local-8082
        url: http://localhost:5003/callback
        headers:
           Authorization: [Bearer <an example token>]
        timeout: 1s
        threshold: 10
        backoff: 1s
        disabled: true
      - name: local-8083
        url: http://localhost:8083/callback
        timeout: 1s
        threshold: 10
        backoff: 1s
        disabled: true
```

### start / stop

```
sudo docker start registry
sudo docker stop registry
sudo docker logs --tail=100 -f registry
```

### administration

```
sudo docker run -t -i --rm --volumes-from registry_volume ubuntu /bin/bash
```

## Nginx proxy

Building the image

docker build -t wilelb/nginx .


1. create volume container
sudo docker create --name nginx_volume wilelb/nginx

2. generate certificate

2. configure volume container
sudo docker run -t -i --rm --volumes-from nginx_volume wilelb/nginx /bin/bash
mkdir -p etc/nginx/ssl
vi /etc/nginx/ssl/nginx.ssl.config

```
[req]
prompt=no
default_bits=2048
encrypt_key=no
default_md=sha2
distinguished_name=req_distinguished_name
# PrintableStrings only
string_mask=MASK:0002

[req_distinguished_name]
0.organizationName = <ORGANIZATION>
emailAddress = <EMAIL>
countryName = <COUNTRY>
commonName = <HOSTNAME>
```

```
openssl req -config /etc/nginx/ssl/nginx.ssl.config -new -x509 -days 365 -keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt
```

```
cd /etc/nginx/ssl/
chmod go-r nginx.*
```

3. create server
```
sudo docker create --link registry:registry -p 80:80 -p 443:443 --volumes-from nginx_volume --name nginx wilelb/nginx
sudo docker run -t -i --rm --volumes-from nginx_volume ubuntu /bin/bash
```

Example nginx configuration (See `nginx-no-auth.conf` as well):
```
#
# HTTPS proxy, no authentication
#
upstream docker_registry_servers {
        server registry:5000;
}

server {
        listen 443;
        server_name <HOSTNAME>;

        ssl on;
        ssl_certificate /etc/nginx/ssl/nginx.crt;
        ssl_certificate_key /etc/nginx/ssl/nginx.key;

        ssl_session_timeout 5m;

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers "HIGH:!aNULL:!MD5 or HIGH:!aNULL:!MD5:!3DES";
        ssl_prefer_server_ciphers on;

        location / {
                # disable client body size limit. Prevents "Server error: 413 trying to push ... blob" errors when pushing layers #
		client_max_body_size 0;

                # basic authentication
                auth_basic "Restricted";
                auth_basic_user_file /etc/nginx/.htpasswd;
                #See: https://github.com/docker/distribution/issues/452
                more_set_headers 'Docker-Distribution-Api-Version: registry/2.0';

                proxy_pass         http://docker_registry_servers/;
                proxy_redirect     off;
                proxy_set_header   Host $host;
                proxy_set_header   X-Real-IP $remote_addr;
                proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header   X-Forwarded-Host $server_name;
                #Prevents docker push from switching to http for layer uploads
                proxy_set_header   X-Forwarded-Proto https;       
        }
}
```

## Expected result

If both the registry and nginx containers have been started, the expected result
should look something similar to the examples below.

Listing the running containers, `sudo docker ps`, should produce the following:
```
CONTAINER ID        IMAGE                 COMMAND                CREATED             STATUS              PORTS                          NAMES
ddfa86b515f2        clarin/nginx:latest   "/bin/sh -c 'service   About an hour ago   Up About an hour    80/tcp, 0.0.0.0:443->443/tcp   nginx
348d482634e1        registry:2.0          "registry /etc/regis   6 hours ago         Up 6 hours          127.0.0.1:5000->5000/tcp       registry
```

Listing all containers, `sudo docker ps -a`, should produce the following:
```
CONTAINER ID        IMAGE                 COMMAND                CREATED             STATUS              PORTS                          NAMES
ddfa86b515f2        clarin/nginx:latest   "/bin/sh -c 'service   About an hour ago   Up About an hour    80/tcp, 0.0.0.0:443->443/tcp   nginx
3ad7c63f0b8f        clarin/nginx:latest   "/bin/sh -c 'service   6 hours ago                                                            nginx_volume
348d482634e1        registry:2.0          "registry /etc/regis   6 hours ago         Up 6 hours          127.0.0.1:5000->5000/tcp       registry
180744c3b259        registry:2.0          "registry cmd/regist   6 hours ago                                                            registry_volume
```

## Generating user accounts

Start a container with the nginx_volume mounted:
```
sudo docker run -ti --rm --volumes-from nginx_volume clarin/nginx /bin/bash
```

To create the initial .htpasswd file:
```
htpasswd -c /etc/nginx/.htpasswd <username> <password>
```

Add additional accounts to the .htpasswd file:
```
htpasswd /etc/nginx/.htpasswd <newusername> <newpassword>
```

To finish up just exit the container. Since we specified the `--rm` flag the container
is nicely cleaned up upon exit.

## Configure docker daemon to trust the private registry certificate

Ssh into the boot2docker vm and switch to the root user:
```
boot2docker ssh
sudo su
```

Install the certificate for the docker daemon:
```
mkdir -p /etc/docker/certs.d/docker.clarin.eu
echo -n | openssl s_client -connect docker.clarin.eu:443 | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > /etc/docker/certs.d/docker.clarin.eu/ca.crt
```

Install the certificate:
```
/usr/local/etc/ssl/certs# ln -s /etc/docker/certs.d/docker.clarin.eu/ca.crt docker.clarin.eu.crt
openssl x509 -hash -in docker.clarin.eu.crt
ln -s docker.clarin.eu.crt c6359835.0
```

If you want to install the certificate for another private registry you can replace "docker.clarin.eu" with the fully qualified domain name of that host.

## Links
* https://docs.docker.com/registry/
* https://docs.docker.com/registry/deploying/
* https://github.com/docker/distribution/blob/master/docs/configuration.md
* https://github.com/docker/distribution/blob/master/docs/spec/auth/token.md

* https://github.com/docker/distribution/issues/489
* https://github.com/docker/distribution/issues/397
* https://github.com/docker/docker-registry/issues/852
* http://grokbase.com/t/gg/docker-user/154raty2c1/docker-registry-v2-with-nginx-auth-fails

# Using the private docker registry

## log in

```
docker login docker.clarin.eu
```
And supply your username, password and optionally your email.

## pushing

Tag a docker image with the private registry information:
```
docker tag wilelb/unity-idm docker.clarin.eu/unity-idm
```

Push this tag to the private registry:
```
docker push docker.clarin.eu/unity-idm
```

## pulling

Just pull the image for later use:
```
docker pull docker.clarin.eu/unity-idm
```

Run the container, implicitly pulling the image if it is not available yet:
```
docker run docker.clarin.eu/unity-idm
```