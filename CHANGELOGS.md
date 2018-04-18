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