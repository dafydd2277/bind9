# Plug and Play DNS Container

This is a mostly Plug-and-Play installation of [ISC][ref001]'s
[BIND9][ref002] container, using [docker-compose][ref003].

[ref001]: https://www.isc.org/
[ref002]: https://hub.docker.com/_/bind9
[ref003]: https://docs.docker.com/compose/


## How to use

1. Install Docker or Podman as appropriate for your underlying
operating system. This fork was developed on [CentOS 8][ref011] using
[docker][ref012].
1. Add or uncomment these lines in `/etc/rsyslog.conf`.
    ```
    module(load="imudp") # needs to be done just once
    input(type="imudp" port="514")

    module(load="imtcp") # needs to be done just once
    input(type="imtcp" port="514")
    ```
    If you want to limit `rsyslog` to only listen to specific
    interfaces, change the `input` lines to something like this:
    ```
    input(type="imudp" port="514" address="192.168.1.1")
    input(type="imtcp" port="514" address="192.168.1.1")
    ```
    See [here for UDP][ref015], and [here for TCP][ref016].
1. Create [a location to hold the container files][ref013].
    ```
    mkdir /srv/containers
    cd /srv/containers
    git clone git@github.com:dafydd2277/bind9.git
    cd bind9
    ```
1. Create the group and user, and prevent anyone from logging in as the
user. IMPORTANT: The GID and UID set here are required in order to
match the GID and UID used inside the container. Without this match,
the BIND9 container will not be able to write back to the `source`
filesystems. Using a different UID or GID will cause the container to
fail.

    ```bash
    groupadd \
      --gid 106 \
      bind
    
    
    useradd \
      --comment "User for bind container" \
      --gid 106 \
      --home-dir /srv/containers/bind9 \
      --no-create-home \
      --no-user-group \
      --shell /bin/true \
      --uid 105 \
      bind
    
    chage -E 0 bind
    
    chown -Rv bind:bind /srv/containers/bind9
    ```
1. Copy the rsyslog file from the project and restart `rsyslog`.
    ```
    cp -i ./etc/rsyslog.d/10-containers.conf /etc/rsyslog.d/
    systemctl restart rsyslog
    ```
    The `-i` switch is to force a confirmation question if a file
    called `10-containers.conf` already exists in that directory. If
    that is the case, merge the two files as appropriate for your
    environment.
1. Copy the logrotate file from the project.
    ```
    cp -i ./etc/logrotate.d/containers /etc/logrotate.d/
    ```
    The `-i` switch is to force a confirmation question if a file
    called `containers` already exists in that directory. If that is
    the case, merge the two files as appropriate for your environment.
1. Modify `.env` to suit your environment.
1. Modify `compose.yaml` to suit your environment. Note that you have
to uncomment the appropriate `ports` entry. If you want to listen to
more than one interface, but not all interfaces, you can c/p several
copies of the second option, modifying the `${s_host_internal_ip}`
value as appropriate.
1. [Modify][ref014] [the named.conf][ref015] file in `./etc/bind/` to
suit your environment.
    * Pay particular attention to the opportunty to set up an RNDC
    key to allow you to have `dhcpd` automatically update `named` to
    add DNS entries as new hosts join the subnet. Execute these
    commands to generate the key.
        ```
        cd /tmp
        dnssec-keygen -a hmac-md5 -b 256 -n USER dhcpupdate
        egrep '^Key' Kdhcpupdate.*.private | cut -d' ' -f2
        ```
        Replace the `FIXME` in the `./etc/bind/dhcpupdate.key` file
        with the resulting hash. You'll use the same hash in
        `dhcpd.conf` to give `dhcpd` the privilege of updating DNS
        records automatically. Don't forget to `rm -f Kdhcpupdate*`
        when you're done.
    * Repeat the process to create a key for `./etc/bind/rndc.key`.
    This will allow you to dynamically manage `named` through
    [rndc][ref016]. Don't use the same key for both functions.
1. [Modify][ref017] [the zone files][ref018] in `./var/lib/bind/` to
suit your environment.
1. Execute
    ```
    grep -r 'FIXME' *
    ```
    to verify you have found all of the customizations you need to make
    in your configuration and zone files. Correct those lines to suit
    your environment.
1.  As root, execute `docker-compose up -d`.


[ref011]: https://centos.org/
[ref012]: https://docs.docker.com/engine/install/centos/ 
[ref013]: https://refspecs.linuxfoundation.org/FHS_3.0/fhs/ch03s17.html
[ref014]: http://www.zytrax.com/books/dns/ch6/
[ref015]: https://bind9.readthedocs.io/en/v9_16_6/reference.html#configuration-file-elements
[ref016]: https://linuxconfig.org/configure-rndc-key-for-bind-dns-server-on-centos-7
[ref017]: https://arstechnica.com/gadgets/2020/08/understanding-dns-anatomy-of-a-bind-zone-file/
[ref018]: https://bind9.readthedocs.io/en/v9_16_6/reference.html#zone-file


## Copyright & License

This project is &copy;2021 David Barr <dafydd@dafydd.com>, and licensed
under [CC BY-SA 4.0][ref051].

![CC](https://creativecommons.org/presskit/icons/cc.svg?ref=chooser-v1)
![CC BY](https://mirrors.creativecommons.org/presskit/icons/by.svg?ref=chooser-v1)
![CC SA](https://mirrors.creativecommons.org/presskit/icons/sa.svg?ref=chooser-v1)

[ref051]: https://creativecommons.org/licenses/by-sa/4.0/?ref=chooser-v1

