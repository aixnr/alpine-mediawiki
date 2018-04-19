## Changelogs

**17 Apr 2018 (Tuesday)** Build initialization.

To install Docker:

```
# Download & install Docker
wget -qO- https://get.docker.com/ | sh

# Adding $USER to docker group
sudo usermod -aG docker $USER
```

Run build for `Dockerfile-mwiki` Dockerfile, tag with `alpine-wiki`

```
docker build -f Dockerfiles/Dockerfile-mwiki -t alpine-mwiki .
```

This Docker image has user `www` with UID of `1000` to match the typical first user on host machine. This would make it easier for volume mounting (installing extension, configuring stuff, etc).

To run and inspect:

```
docker run --name alpine-wiki --rm -p 80:80 -p 9001:9001 -it alpine-mwiki
```

The port `80` is used by the `nginx`, while port `9001` is used for the `supervisord` web UI (password protected, see `configs/supervisord.conf`). The `-it` flag for interactive mode. Pass `-d` flag for daemon mode.

If there is a problem and need to access the container, run this:

```
docker exec -it alpine-wiki /bin/bash
```

Run this Docker image with volume `-v` flag, then mount that volume with `bindfs`. Why do this? Well, it is easier to place the `LocalSettings.php` and eventual configuration to add more extensions in the future. With `bindfs`, I can mount the root-limited directory at `/var/lib/docker/volumes/` to `$(pwd)/data` while not royally messing up with the permission since `bindfs` is smart enough to mirror the changes.

Then proceed with MediaWiki web installation. At the end of the installation process, download `LocalSettings.php` and place it at `$(pwd)/data/LocalSettings.php`, then assign the readable permission.

```
# Install bindfs
sudo apt install bindfs

# Run the container, set to destroy after exit
docker run --name alpine-wiki -v wiki-data:/var/www/mediawiki --rm -p 80:80 -p 9001:9001 -it alpine-mwiki

# bind from volume to data directory
sudo bindfs /var/lib/docker/volumes/wiki-data/_data data

# Copy LocalSettings.php & set 775 permission
chmod 775 LocalSettings.php
```

This trick is possible because during the creation of this Docker image, the `UID` of user `www` is mapped to `1000`, which is the typical first user on my linux host.

**18 Apr 2018 (Wed)** Tweaking Mediawiki image.

First change: so instead of download the `*.tar.gz` archive, I decided to switch to Git. Current stable branch is `REL1_30`, which is specified in the `Dockerfile-mwiki`. Composer is needed for this to work.

**18 Apr 2018 (Wed)** Database with MySQL/MariaDB.

Create a network (since `link` is a legacy feature to connect our containers. Then run a MariaDB v10 container and the MediaWiki container on the same network.

```
# Define network
docker network create mediawiki

# Run the database container with --network flag.
docker run -d -e MYSQL_ROOT_PASSWORD=password --network mediawiki --name mariadb mariadb:10

# Run MediaWiki in the same network.
docker run --name alpine-wiki -v wiki-data:/var/www/mediawiki --network mediawiki --rm -p 80:80 -p 9001:9001 -it alpine-mwiki

# To test, exec into alpine-wiki container and ping mariadb
docker exec -it alpine-wiki /bin/bash
ping mariadb
```

Problem: during web installation, the MediaWiki does not see MariaDB. To fix this, it is probably wise to use `docker-compose.yml` to better retain sanity. Make sure that the image `alpine-mwiki:latest` is actually available.

```
version: '3'
services:
  mediawiki:
    image: alpine-mwiki
    restart: always
    ports:
      - 80:80
      - 9001:9001
    links:
      - mariadb
    volumes:
      - wikidata:/var/www/mediawiki
  mariadb:
    image: mariadb:10
    restart: always
    environment:
      MYSQL_DATABASE: my_wiki
      MYSQL_USER: my_wiki
      MYSQL_PASSWORD: my_password
      ALLOW_EMPTY_PASSWORD=yes

volumes:
  wikidata
```

**Abort this installation**. Somehow, MariaDB and MySQL were giving me headache. Specifically, the MediaWiki web installer just could not see the MySQL/MariaDB. But then just after trying it once with Postgres, I got it connected.

**18 Apr 2018 (Wed)** Database with Postgres.

This is the `docker-compose.yml`

```
version: '3.2'
services:
  mediawiki:
    image: alpine-mwiki
    restart: always
    ports:
      - 80:80
      - 9001:9001
    links:
      - pg
    depends_on:
      - pg
    volumes:
      - wikidata:/var/www/mediawiki
    environment:
      MEDIAWIKI_DB_TYPE: postgres
      MEDIAWIKI_DB_HOST: pg
      MEDIAWIKI_DB_USER: wiki
      MEDIAWIKI_DB_PASSWORD: wiki
      MEDIAWIKI_DB_NAME: wiki
  pg:
    image: postgres:10-alpine
    restart: always
    environment:
      POSTGRES_DB: wiki
      POSTGRES_USER: wiki
      POSTGRES_PASSWORD: wiki
    volumes:
      - wikidb:/var/lib/postgresql/data

volumes:
  wikidata:
  wikidb:
```

During the installation, the location of the Postgres is `pg`, not `localhost`. Then mount using `bindfs`

```
# Create directory first.
mkdir data_pg

# Bind, mirror to fera since it is owned by root inside the container.
sudo bindfs --mirror=fera /var/lib/docker/volumes/alpinemediawiki_wikidb/_data data_pg
```

Do the same for `/var/www/mediawiki`

```
# Create directory first.
mkdir data_wiki

# bind, no need to map user because both have same UID.
sudo bindfs /var/lib/docker/volumes/alpinemediawiki_wikidata/_data data_wiki
```

**18 Apr 2018 (Wed)** NodeJS & Parsoid.

It appears that Parsoid's `config.yaml` requires hardcoded URL. During build, the `config.yaml` is supplied from `configs/parsoid-config.yaml`. To change the config of an already running Parsoid container, there are 2 ways: mount Parsoid's `config.yaml` by explicitly defining it in `docker-compose.yaml` or specify a Docker volume with `-v` then later mount it to edit the `config.yaml`.

Build the image:

```
docker build -f Dockerfiles/Dockerfile-parsoid -t alpine-parsoid .
```

Run to debug (output to terminal, destroy upon exit).

```
docker run --name parsoid -it -p 8000:8000 --rm alpine-parsoid
```

The hardcoded configuration is pointing to `http://192.168.56.101/api.php`. This is the IP address of my VM linux box. Change it before build. In `LocalSettings.php`, here's the directive:

```
$wgVirtualRestConfig['modules']['parsoid'] = array(
  'url' => 'http://192.168.56.101:8000',
);
```

No restart needed. Parsoid runs at 120 MB. Not too bad. Postgres runs at 6 MB, main container runs at 56 MB. So roughly the whole setup runs less than 200 MB.