## MediaWiki, Parsoid, and PostgreSQL with Docker

This is an attempt to use Alpine 3.7 as the base image for Mediawiki 1.30 with PHP v7.1.16 (with FPM) and Nginx. There is another container for Parsoid (insert version here) that runs with NodeJS (insert version here). As the database backend (specified in `docker-compose.yml`), the database backend is PostgreSQL v10 (Alpine).

### The Images

Clone this repository. Then run `docker build` to create the `alpine-mwiki` image.

```
docker build -f Dockerfiles/Dockerfile-mwiki -t alpine-mwiki .
```

During this build, the code will (briefly):

1. Install the base system packages (also with Nginx, Git, etc), installing PHP v7 and all the PHP packages required to run MediaWiki.
2. Install Composer to manager dependencies.
3. Clone the MediaWiki branch `REL1_30` and install it.
4. Expose ports 80 for Nginx and 9001 for `supervisord` web UI.
5. Copy configuration files for `supervisord`, Nginx, and PHP7-FPM.
6. Add user `www` for PHP7-FPM with UID of `1000` and GID of `1000`.
7. Set `ENTRYPOINT` with `supervisord`.

### The Containers

To retain sanity, here's the content for `docker-compose.yml`

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

### Mount Volume with BindFS

At the end of web installation of the MediaWiki, user is prompted to download the file `LocalSettings.php`. That file must go into the MediaWiki root installation directory (the `/var/www/mediawiki` inside the container). As previously mentioned, this root installation folder is mounted as a volume by Docker with the `-v` flag during run. This Docker volume is located at (insert location here), with root-only access (Docker's default). 

However, data written within that folder is owned by user `www` of the container with the UID of `1000`. This was configured during the  image build process (see `Dockerfiles/Dockerfile-mwiki`).

Here we use the `bindfs` to mount this volume into `./data_wiki` directory. `BindFS` allows for (something here).

```
# Create directory.
mkdir data_wiki

# Mount with bindfs
sudo bindfs /var/lib/docker/volumes/alpinemediawiki_wikidata/_data data_wiki
```

It is slightly different with `postgres:10-alpine`. By default, `postgres` runs as `root`, hence there is a slight modification to our command here.

```
# Create directory first.
mkdir data_pg

# Bind, mirror to fera since it is owned by root inside the container.
sudo bindfs --mirror=fera /var/lib/docker/volumes/alpinemediawiki_wikidb/_data data_pg
```

This will map the Docker volume correctly with user `fera` (depends on the `$USER` you have on host machine, mine is `fera`). By issuing `ls -l data_pg`, the file permission is assigned to `fera` instead of `root`.

### Notes

1. I tried using MariaDB at first. I could not get the MediaWiki to talk to MariaDB instance after trying couple of times. Miraculously, I tried once with PostgreSQL and it worked without fiddling too much.
2. Also, I am using `postgres:10-alpine` (official). It is extremely small at 39.5 MB for the Docker image. MariaDB (Debian) is at 396 MB.
3. Please run the Nginx and PHP-FPM with daemonize turned off. Else, it won't work. This has something to do with the fact that `supervisord` is daemonizing them.
4. BindFS is pretty cool.

### TODO

- [ ] Change download method from `wget` to `git clone`.
- [ ] Compose-related.
    - [X] Attempt at creating baseline `docker-compose.yml`.
    - [ ] Mount logo as ReadOnly in Compose.
- [ ] Create Alpine 3.7 image for Parsoid.

### Acknowledgements

Thanks to (insert list here).