## MediaWiki, Parsoid, and MariaDB with Docker

This is an attempt to use Alpine 3.7 as the base image for Mediawiki 1.30 with PHP v7 (with FPM) and Nginx. There is another container for Parsoid (insert version here) that runs with NodeJS (insert version here). As the database backend (specified in `docker-compose.yml`), the database backend is MariaDB (insert version here).

### The Images

Clone this repository. Then run `docker build` to create the `alpine-mwiki` image.

```
docker build -f Dockerfiles/Dockerfile-mwiki -t alpine-mwiki .
```

During this build, the code will (briefly):

1. Install the base system packages (also with Nginx, Git, etc), installing PHP v7 and all the PHP packages required to run MediaWiki.
2. Install Composer to manager dependencies.
3. Git clone the MediaWiki branch `REL1_30`.
4. Expose ports 80 for Nginx and 9001 for `supervisord` web UI.
5. Copy configuration files for `supervisord`, Nginx, and PHP7-FPM.
6. Add user `www` for PHP7-FPM with UID of `1000` and GID of `1000`.
7. Setting `ENTRYPOINT` with `supervisord`.

### TODO

- [ ] Add `docker-compose.yml`.
- [ ] Create Alpine 3.7 image for Parsoid.