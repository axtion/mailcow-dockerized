# mailcow-dockerized

mailcow dockerized comes with 11 containers linked in a mailcow network:
Dovecot, Memcached, Redis, MariaDB, PowerDNS Recursor, PHP-FPM, Postfix, Nginx, Rmilter, Rspamd and SOGo.

All configurations were written with security in mind.

### Exposed ports:

| Name              | Service      | Hostname, Alias                | External bindings                            | Internal bindings              |
|:------------------|:-------------|:-------------------------------|:---------------------------------------------|:-------------------------------|
| postfix-mailcow   | Postfix      | ${MAILCOW_HOSTNAME}, postfix   | 25/tcp, 465/tcp, 587/tcp                     | 588/tcp                        |
| dovecot-mailcow   | Dovecot      | ${MAILCOW_HOSTNAME}, dovecot   | 110/tcp, 143/tcp, 993/tcp, 995/tcp, 4190/tcp | 24/tcp, 10001/tcp              |
| nginx-mailcow     | Nginx        | nginx                          | 443/tcp                                      | 80/tcp, 8081/tcp               |
| pdns-mailcow      | PowerDNS     | pdns                           | -                                            | 53/udp                         |
| rspamd-mailcow    | Rspamd       | rspamd                         | -                                            | 11333/tcp, 11334/tcp           |
| mariadb-mailcow   | MariaDB      | mysql                          | -                                            | 3306/tcp                       |
| rmilter-mailcow   | Rmilter      | rmilter                        | -                                            | 9000/tcp                       |
| phpfpm-mailcow    | PHP FPM      | phpfpm                         | -                                            | 9000/tcp                       |
| sogo-mailcow      | SOGo         | sogo                           | -                                            | 9000/tcp                       |
| redis-mailcow     | Redis        | redis                          | -                                            | 6379/tcp                       |
| memcached-mailcow | Memcached    | memcached                      | -                                            | 11211/tcp                      |

All containers share a network ${MAILCOW_NETWORK} (name can be changed, but remove all containers and rebuild them after changing).
IPs are dynamic and taken from subnet ${DOCKER_SUBNET}.

## Installation

1. You need Docker. Most systems can install Docker by running `wget -qO- https://get.docker.com/ | sh`

2. Clone this repository and configure `mailcow.conf`, do not use special chars in passwords in this file (will be fixed soon).
It is almost always enough to just change the hostname.

3. Run `./build-all.sh` - select `Y` when asked to reset the admin password.

Done.

You can now access https://${MAILCOW_HOSTNAME} with the default credentials `admin` + password `moohoo`.

## Configuration after installation

### Rspamd UI access
If you want to use Rspamds web UI, you need to set a Rspamd controller password:

```
# Generate hash
docker exec -it rspamd-mailcow rspamadm pw
```

Replace given hash in data/conf/rspamd/override.d/worker-controller.inc:
```
enable_password = "myhash";
```

Restart rspamd:
```
docker restart rspamd-mailcow
```

Open https://${MAILCOW_HOSTNAME}/rspamd in a browser.

### SSL (and: How to use Let's Encrypt)
mailcow dockerized generates a CA named "mailcow" with a self-signed server certificate in `data/assets/ssl` via `000-build-certs.sh`.

Get the certbot client:
```
wget https://dl.eff.org/certbot-auto -O /usr/local/sbin/certbot && chmod +x /usr/local/sbin/certbot
```

Please disable applications blocking port 80 and run certbot:
```
certbot-auto certonly \
	--standalone \
	--standalone-supported-challenges http-01 \
	-d ${MAILCOW_HOSTNAME} \
	--email you@example.org \
	--agree-tos
```

Create hard links to the full path of the new certificates. Assuming you are still in the mailcow root folder:
```
mv data/assets/ssl/cert.{pem,pem.backup}
mv data/assets/ssl/key.{pem,pem.backup}
ln $(readlink -f /etc/letsencrypt/live/${MAILCOW_HOSTNAME}/fullchain.pem) data/assets/ssl/mail.crt
ln $(readlink -f /etc/letsencrypt/live/${MAILCOW_HOSTNAME}/privkey.pem) data/assets/ssl/mail.key
```

Restart containers which use the certificate:
```
docker restart postfix-mailcow
docker restart dovecot-mailcow
docker restart nginx-mailcow
```

When renewing certificates, run the last two steps (link + restart) as post-hook in certbot.

## Special usage
### build-*.files

(Re)build a container:
```
./n-build-$name.sh 
```
**:exclamation:** Any previous container with the same name will be stopped and removed.
No persistent data is deleted at any time.
If an image exists, you will be asked wether or not to repull/rebuild it.

Build files are numbered "nnn" for dependencies.

### Logs

You can use docker logs $name for almost all containers. Only rmilter does not log to stdout. You can check rspamd logs for rmilter responses.

### MariaDB

Connect to MariaDB database:
```
./n-build-sql.sh --client
```

Init schema (will also be installed when running `./n-build-sql.sh` without parameters):
```
./n-build-sql.sh --init-schema
```

Reset mailcow admin to `admin:moohoo`:
```
./n-build-sql.sh --reset-admin
```

Dump database to file backup_${DBNAME}_${DATE}.sql:
```
./n-build-sql.sh --dump
```

Restore database from a file:
```
./n-build-sql.sh --restore filename
```

### Redis

Connect to redis database:
```
./n-build-redis.sh --client
```

### Some examples

Use rspamadm:
```
docker exec -it rspamd-mailcow rspamadm --help
```

Use rspamc:
```
docker exec -it rspamd-mailcow rspamc --help
```

Use doveadm:
```
docker exec -it dovecot-mailcow doveadm
```

### Remove persistent data

MariaDB:
```
docker stop mariadb-mailcow
docker rm mariadb-mailcow
rm -rf data/db/mysql/*
./n-build-sql.sh
```

Redis:
```
# If you feel hardcore:
docker stop redis-mailcow
docker rm redus-mailcow
rm -rf data/db/redis/*
./n-build-redis.sh

## It is almost always enough to just flush all keys:
./n-build-redis client
# FLUSHALL [ENTER]
```