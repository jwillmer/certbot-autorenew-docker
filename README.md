## Certbot Auto Renewal

**Available as docker image on [GitHub Package Registry](https://github.com/jwillmer/certbot-autorenew/pkgs/container/certbot-autorenew).**

This image runs [`certbot`](https://certbot.eff.org/) under the hood to automate issuance and renewal of letsencrypt certificates.

Initial certificate requests are run at container first launch, once the image responds on a specified health check url.

Then certificates validity is checked at 02:00 on every 7th day-of-month from 1 through 31, and certificates are renewed only if expiring in less that 28 days, preventing from being rate limited by letsencrypt.

Issued certificates are made available in the container's `/certs` directory which can be mounted on the docker host or as a docker volume to make them available to other applications.

### Requirements

- docker
- docker-compose

### Configure the image's run parameters
 Adapt the provided `docker-compose.yml` file to fit your requirements. The required/optional parameters are described here after:

#### Build or fetch the docker image

- Either build image with provided docker file
- -or- fetch the image from GitHub registry at [`ghcr.io/jwillmer/certbot-autorenew:latest`](https://github.com/jwillmer/certbot-autorenew/pkgs/container/certbot-autorenew)

#### Ports
The software in the docker container exposes internally the `443` port, which you should expose back on the docker host with no translation, such as in `"443:443"`

#### Volumes
The following volumes of interest can be mounted on the docker host or as docker volumes:
- **/certs** : location of certificates generated by letsencrypt, this is the main directory of interest to expose to your application
- **/etc/letsencrypt** : location of letsencrypt install dir (optional, for debug purposes)
- **/var/log/letsencrypt** : location of letsencrypt logs (optional, for debug purposes)
- **/scripts/pre-hook.sh** : script location that gets executed before certificates get issued. Will be called with all domains as first argument.
- **/scripts/post-hook.sh** : script location that gets executed before certificates get issued. Will be called with all domains as first argument.


#### Environment variables:
- **WEBROOT** : (optional) path to the host's web server root. If provided, letsencrypt will use the given existing web server to request and validate the certificates. If not provided, letsencrypt will launch it's own web server for this purpose
- **PLUGIN** : (optional, defaults to `standalone`) A `certbot` [plugin](https://certbot.eff.org/docs/using.html#dns-plugins) to use (e.g. `manual`). A list of installed plugins can be found in the `Dockerfile`.
- **PREFERRED_CHALLENGES** : (optional, defaults to http-01) A sorted, comma delimited list of the preferred challenge to use during authorization with the most preferred challenge listed first (eg. "dns-01" or "tls-alpn-01,http-01,..").
- **CUSTOM_ARGS** : (optional) Additional `certbot` command-line options (e.g. `--redirect`), refer to certbot [documentation](https://certbot.eff.org/docs/using.html#certbot-command-line-options)
- **LOGFILE** : (optional) path of a file where to write the logs from the certificate request/renewal script. When not provided both stdout/stderr are directed to console which is convenient when using a docker log driver
- **DEBUG** : (optional) whether to run letsencrypt in debug mode, refer to certbot [documentation](https://certbot.eff.org/docs/using.html#certbot-command-line-options)
- **STAGING** : (optional) whether to run letsencrypt in staging mode, refer to certbot [documentation](https://certbot.eff.org/docs/using.html#certbot-command-line-options)
- **DOMAINS** : space separated list of comma separated subdomains to register the certificate with, for example:
  - `my.domain.com`
  - `sub.domain1.com,sub.domain2.com`
  - `my.other.domain.com sub.domain1.com,sub.domain2.com`
- **EMAIL** : email of the certificates supplicant
- **CONCAT** : (optional) whether to concatenate the full chain of the certificate authority with the certificate's private key. This is required for example for haproxy. Otherwise the full chain and private key are kept in separate files which is required for example for nginx and apache
- **HEALTH_CHECK_URL** : (optional) a publicly accessible health check url on which the software in the docker container can verify and wait for the docker host to be up and ready to accept connections
- **RUN_AT_START** : (optional) Executes the cert request imediately instead of waiting for the next cron execution
- **PPID** : (optional) Set process ID

#### Example
In the `docker-compose.yml` file is an example to use the image with the cloudflare plugin. The below code is the minimal configuration for this image:

```yml
version: '2'

services:
  certbot:
    build: .
    image: ghcr.io/jwillmer/certbot-autorenew:latest
    container_name: certbot
    volumes:
      - ~/certificates:/certs
      - ~/certbot/logs:/var/log/letsencrypt
    environment:
      - LOGFILE=/var/log/letsencrypt/certrenewal.log
      - RUN_AT_START=true
      - DOMAINS=*.test.example.com
      - EMAIL=info@example.com
      - DEBUG=true
      - STAGING=true
```

### Docker Logs
When using a docker logging driver, the `LOGFILE` environment variable should not be set to make sure all the container logs (stdout/stderr) are directed to the console, and hence to the logging driver.

An example is provided for **aws logging driver**. This should be
```yml
version: '2'

services:
  certbot:
    # ...
    environment:
      # ...
      # LOGFILE should not be set when working with a docker logging driver
      # - LOGFILE=/var/log/letsencrypt/certrenewal.log
    logging:
      driver: "awslogs"
      options:
        awslogs-region: "${AWS_REGION}"
        awslogs-group: "hooly-search"
        awslogs-stream: "letsencrypt"
```

### Build / run the container

#### Building
Build and run the container as follows:
```sh
docker-compose build
docker-compose up -d
```

#### Running image from dockerhub
```sh
docker-compose up -d
```
