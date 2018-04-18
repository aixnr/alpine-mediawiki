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
3. Clone the MediaWiki branch `REL1_30` and install it.
4. Expose ports 80 for Nginx and 9001 for `supervisord` web UI.
5. Copy configuration files for `supervisord`, Nginx, and PHP7-FPM.
6. Add user `www` for PHP7-FPM with UID of `1000` and GID of `1000`.
7. Set `ENTRYPOINT` with `supervisord`.

### The Containers

Actually running stuff here.

### Mount Volume with BindFS

At the end of web installation of the MediaWiki, user is prompted to download the file `LocalSettings.php`. That file must go into the MediaWiki root installation directory (the `/var/www/mediawiki` inside the container). As previously mentioned, this root installation folder is mounted as a volume by Docker with the `-v` flag during run. This Docker volume is located at (insert location here), with root-only access (Docker's default). 

However, data written within that folder is owned by user `www` of the container with the UID of `1000`. This was configured during the  image build process (see `Dockerfiles/Dockerfile-mwiki`).

Here we use the `bindfs` to mount this volume into `./data` directory. `BindFS` allows for (something here).

### TODO

- [ ] Change download method from `wget` to `git clone`.
- [ ] Add `docker-compose.yml`.
- [ ] Create Alpine 3.7 image for Parsoid.

### Acknowledgements

Thanks to (insert list here).