# CBSD Mirrors

## Intro

The CBSD project uses images ( ISO and CLOUD ) referenced by the project's [base profiles](https://github.com/cbsd/cbsd-vmprofiles).
Since images on the original sites tend to disappear after a while (especially if it's a rolling release model), to improve the quality of service, the CBSD project maintains its own mirror infrastructure (you can take part in it and become a contributor to the project - see below).

These mirrors store up-to-date images referenced by the latest versions of profiles, and this is an additional safety net when the original server has already deleted the old image, but CBSD profiles still reference it.

You can [create your own mirror](#creating-your-own-mirror) and use it as/for:

- [together with other mirrors or exclusively/forcefully (for example, force all your hosts to look only at your mirror)](#using-your-own-private-mirror);
- [use a mirror privately](#using-your-own-private-mirror) or (if you have a good channel) [add your mirror to the general index and make users in your region happy](#publishing-your-mirror-for-cbsd-project) (especially if the existing mirrors are located far from you)](#using-your-own-private-mirror);
- you want to be able to use old images - since the original CBSD mirrors are limited in capacity, we cannot store large files forever and usually with each new profile update old images are deleted. You can remove the `--delete` flag when synchronizing via rsync and your mirrors will always store old images, even if the original source no longer has them.

Some statistics in graphs (updated monthly, graph since the beginning of the year):

- distribution popularity based on the number of successful requests to the CBSD mirror (the graph only shows the general trend, since this is data from single (origin) mirror).

![mirror-distro-family-stats.png](https://convectix.com/img/mirror-distro-family-stats.png?raw=true)

Statistics on the number of files and volumes (if you plan to make a full mirror, keep in mind that the volumes will most likely grow - new profiles appear):

![mirror-stats.png](https://convectix.com/img/mirror-stats.png?raw=true)

## Creating your own mirror

Images are synchronized using the [RSYNC](https://rsync.samba.org/) utility:

- Origin-server for ISO images: **rsync://mirror.convectix.com/iso/**
- Origin-server for CLOUD images: **rsync://mirror.convectix.com/cloud/**
- Origin-server for base/rootfs and ([marketplace](https://marketplace.convectix.com) containers): **WIP**

<details>
  <summary>Example settings FreeBSD-based environment</summary>

:bangbang: | :Info: You can get a ready-made container with a web service and a cron task for creating a CBSD mirror from the [marketplace](https://marketplace.convectix.com/#cbsdmirror) !
:---: | :---


---


Step-by-step setup (on FreeBSD) of the mirror with periodic synchronization via RSYNC (http://rsync.samba.org/) in crontab(5) (http://man.freebsd.org/crontab/5):

1) Install packages:

```
pkg install -y rsync nginx
```

2) Activate nginx services:

```
sysrc nginx_enable="YES"
```

3) Create _/usr/local/www/cbsd-mirror_ directory where we will save ISO images, create a log file for rsync and set the right permissions for the **nobody** user, from which we will synchronize:

```
mkdir -p /usr/local/www/cbsd-mirror/iso /usr/local/www/cbsd-mirror/cloud /var/log/nginx
touch /var/log/cbsd_mirror_iso.log /var/log/cbsd_mirror_cloud.log
chown -R nobody:nobody /usr/local/www/cbsd-mirror /var/log/cbsd_mirror_iso.log /var/log/cbsd_mirror_cloud.log
```

4) Correct **nginx.conf**, specifying **server\_name** as correct name of the server (in this example: **cbsd-mirror.example.com**) and set path to root directory, edit /usr/local/etc/nginx/nginx.conf file like this:

```
user nobody;
worker_processes  2;

error_log       /dev/null;
pid             /var/run/nginx.pid;

events {
        use kqueue;
        kqueue_changes  1024;
        worker_connections  1024;
}

http {
        server_tokens off;
        include       mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" ' '$status $body_bytes_sent "$http_referer" ' '"$http_user_agent" "$http_x_forwarded_for"';

        access_log                      /dev/null;
        client_body_buffer_size         32K;
        client_body_timeout             3m;
        client_header_buffer_size       1k;
        client_header_timeout           3m;
        client_max_body_size            20m;
        error_log                       /dev/null;
        gzip                            off;
        keepalive_timeout               8;
        large_client_header_buffers     4 8k;
        log_not_found                   off;
        output_buffers                  1 32k;
        postpone_output                 1460;
        reset_timedout_connection       on;
        send_timeout                    3m;
        sendfile                        on;
        tcp_nodelay                     on;
        tcp_nopush                      on;

        server {
                listen       *:80;
                #listen      [::]:80;   # Enable IPv6;

                server_name     cbsd-mirror.example.com;  # Set valid name here
                access_log      /var/log/nginx/mirror.example.com.acc main;
                error_log       /var/log/nginx/mirror.example.com.err;
                root            /usr/local/www/cbsd-mirror;
        }
}
```

5) Create an entry in crontab for `nobody` user with the rsync call for 14/15 minutes through lockf to stop duplication of processes

```
cat > /var/cron/tabs/nobody <<EOF
SHELL=/bin/sh
PATH=/etc:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin
*/14    *       *       *       *       /usr/bin/lockf -s -t0 /tmp/cbsd_mirror_iso.lock /usr/local/bin/rsync -a --delete rsync://mirror.convectix.com/iso/ /usr/local/www/cbsd-mirror/iso/ > /var/log/cbsd_mirror_iso.log 2>&1
*/15    *       *       *       *       /usr/bin/lockf -s -t0 /tmp/cbsd_mirror_cloud.lock /usr/local/bin/rsync -a --delete rsync://mirror.convectix.com/cloud/ /usr/local/www/cbsd-mirror/cloud/ > /var/log/cbsd_mirror_cloud.log 2>&1
EOF

chmod 0600 /var/cron/tabs/nobody
```

6) Start WEB service

```
service nginx restart
```
</details>


<details>

  <summary>Example settings Debian-based environment</summary>


---



  Step-by-step setup (on Debian) of the mirror with periodic synchronization via RSYNC (http://rsync.samba.org/) in crontab(5) (http://man.freebsd.org/crontab/5):

1) Install packages:

```
apt-get install -y rsync nginx util-linux
```

2) Activate nginx services:

```
systemctl enable nginx
```

3) Create _/var/www/cbsd-mirror_ directory where we will save ISO images, create a log file for rsync and set the right permissions for the **nobody** user, from which we will synchronize:

```
mkdir -p /var/www/cbsd-mirror/iso /var/www/cbsd-mirror/cloud /var/log/nginx
touch /var/log/cbsd_mirror_iso.log /var/log/cbsd_mirror_cloud.log
chown -R nobody:nogroup /var/www/cbsd-mirror /var/log/cbsd_mirror_iso.log /var/log/cbsd_mirror_cloud.log
```

4) Correct **nginx.conf**, specifying **server\_name** as correct name of the server (in this example: **cbsd-mirror.example.com**) and set path to root directory, edit /etc/nginx/nginx.conf file like this:

```
user www-data;
worker_processes  2;

error_log       /dev/null;
pid             /run/nginx.pid;

events {
        worker_connections  1024;
}

http {
        server_tokens off;
        include       mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" ' '$status $body_bytes_sent "$http_referer" ' '"$http_user_agent" "$http_x_forwarded_for"';

        access_log                      /dev/null;
        client_body_buffer_size         32K;
        client_body_timeout             3m;
        client_header_buffer_size       1k;
        client_header_timeout           3m;
        client_max_body_size            20m;
        error_log                       /dev/null;
        gzip                            off;
        keepalive_timeout               8;
        large_client_header_buffers     4 8k;
        log_not_found                   off;
        output_buffers                  1 32k;
        postpone_output                 1460;
        reset_timedout_connection       on;
        send_timeout                    3m;
        sendfile                        on;
        tcp_nodelay                     on;
        tcp_nopush                      on;

        server {
                listen       *:80;
                #listen      [::]:80;   # Enable IPv6;

                server_name     cbsd-mirror.example.com;  # Set valid name here
                access_log      /var/log/nginx/mirror.example.com.acc main;
                error_log       /var/log/nginx/mirror.example.com.err;
                root            /var/www/cbsd-mirror;
        }
}
```

5) Create an entry in crontab for `nobody` user with the rsync call for 14/15 minutes through lockf to stop duplication of processes

```
cat > /var/spool/cron/crontabs/nobody <<EOF
SHELL=/bin/sh
PATH=/etc:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin
*/14    *       *       *       *       /usr/bin/flock -w0 -x /tmp/cbsd_mirror_iso.lock /usr/bin/rsync -a --delete rsync://mirror.convectix.com/iso/ /var/www/cbsd-mirror/iso/ > /var/log/cbsd_mirror_iso.log 2>&1
*/15    *       *       *       *       /usr/bin/flock -w0 -x /tmp/cbsd_mirror_cloud.lock /usr/bin/rsync -a --delete rsync://mirror.convectix.com/cloud/ /var/www/cbsd-mirror/cloud/ > /var/log/cbsd_mirror_cloud.log 2>&1
EOF

chmod 0600 /var/spool/cron/crontabs/nobody
```

6) Start WEB service

```
systemctl restart nginx
```


</details>

## Example of creating a cbsdmirror container from CBSDfile (FreeBSD jail)

```
cd /tmp
fetch -o /tmp/cbsdfile-recipes.zip https://codeload.github.com/cbsd/cbsdfile-recipes/zip/refs/heads/master
tar xfz cbsdfile-recipes.zip
cd cbsdfile-recipes-master/jail/cbsdmirror/
env CBSD_MIRROR_SYNC_ISO="0" CBSD_MIRROR_SYNC_CLOUD="*/10 * * * *" cbsd up
```

In the last line, we create a container that does NOT sync ISO, but syncs CLOUD images, running every 10 minutes. You can use arguments to override the parameters without editing CBSDfile, for example, create a cbsdmirror container that syncs ISO and CLOUD with an interval of every 14 and 15 minutes respectively (by default) and a fixed IPv4 address, 172.16.0.55:

```
cbsd up ip4_addr="172.16.0.55"
```

After some time after startup, the script inside the container will start synchronization (on the index page http:/172.16.0.55 you can see the statistics) and all images will be available to you via http:/172.16.0.55/iso/ (for ISO) and http://172.16.0.55/cloud/ (for CLOUD), which you can use in CBSD.

## Using your own private mirror

To add your own mirror to the global mirror list, use the ~cbsd/etc/global.conf file to override the iso_site_local= parameter (empty by default). For default parameters, see the contents of the file (~cbsd/etc/defaults/global.conf)[~cbsd/etc/defaults/global.conf]. As an example, let's add the mirror http://172.16.0.55 obtained from the cbsdmirror container (if you are not satisfied with the manual instructions above).

Example of ~cbsd/etc/global.conf file:
```
iso_site_local="http://172.16.0.55/cloud/"
```

After that, when downloading an image on this host, you will see your mirror in the general list of all mirrors.
If your private mirror is fast, then there is a good chance that it will win in preferences and the image will be downloaded from it.

![cbsd_mirror_cust1.png](https://convectix.com/img/cbsd_mirror_cust1.png?raw=true)


If you don't want to leave it to chance and want to unconditionally prefer your mirror and only it (for example, you are in a secure perimeter where Internet access is blocked and it is desirable not to waste time trying to contact public mirrors), then we will need one more parameter as an adjustment in the ~cbsd/etc/global.conf file - `cbsd_fetch_site=`. An example of a new version of ~cbsd/etc/global.conf to use ONLY your mirror:

```
cbsd_fetch_site="iso_site_local"
iso_site_local="http://172.16.0.55/cloud/"
```

Now there is no scanning and mirror sorting stage at all:

![cbsd_mirror_cust2.png](https://convectix.com/img/cbsd_mirror_cust2.png?raw=true)

## Publishing your mirror for CBSD project

Finally, if you have sufficient resources for hosting and a desire to help the CBSD project, you can publish a link to your mirror in the index file, which is located in this repository. To do this, clone this repository and add your servers to the cbsd-iso.txt and cbsd-cloud.txt files and, of course, a contact that can be contacted if any difficulties arise. Then create an MR and when the change is accepted, all CBSD installations will start using your mirror.

In the near future, we plan to launch a health monitoring service for mirrors, which will automatically write out servers from the index in case of prolonged unavailability of mirrors.

1) fork [repo](https://github.com/cbsd/mirrors/fork);

2) make the appropriate changes:
```
git clone git@github.com:XXX/mirrors.git
// edit mirrors/cbsd-iso.txt + mirrors/cbsd-cloud.txt
git add . -A
git commit -am "add new mirrors"
git push
```

3) create MR

## Thanks

The CBSD project is grateful to the [donors](https://www.patreon.com/clonos), thanks to whom you have the opportunity to run a minimal/origin mirror infrastructure.
