---
title: FreeBSD workshop for TongaCERT (2018)
---

**New:** Try the `bsdconfig` command :-)

Questions? You can email me at <kamila@ksp.sk>, I like questions!

# Resources

* FreeBSD Handbook: [PDF](https://download.freebsd.org/ftp/doc/en/books/handbook/book.pdf), [online](https://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/), or [EPUB](https://download.freebsd.org/ftp/doc/en/books/handbook/book.epub)
* FreeBSD downloads page: https://www.freebsd.org/where.html
* programs:
    * SSH client for Windows: [PuTTY](https://putty.org/)
        * you may want to allow root login on your VMs (don't do it in production!) -- look for `PermitRootLogin` in `/etc/ssh/sshd_config`
* specialised FreeBSD-based systems
    * [pfSense](https://www.pfsense.org/): FreeBSD + PF-based firewall with a web interface and extra features out of the box
    * [FreeNAS](http://www.freenas.org/): pre-configured ZFS-based NAS
        * does not really add that much to plain FreeBSD -- especially because you are now all experts on ZFS!
    * [TrueOS](https://www.trueos.org/): FreeBSD with a graphical interface pre-configured with some nice features
        * you can get the same with just plain FreeBSD, but with TrueOS the nice extras are there right after installation
* I will create a fish shell tutorial eventually, the link will be here

# Notes

* Playing with virtual networks in VirtualBox: 
  * use a bridged adapter if you want the VM to get an IP address from the router (like your laptop does)
  * use "internal network" to connect VMs to one another (but not to the outside)
* Very very very useful command I forgot to mention: `bsdconfig` (try it!)

# Exercises

* [PF "homework" exercise](https://trouble.is/~philip/2018-10_TongaCERT/pf.exercise.3.txt)

# What next

## Some suggestions for things which are easy to set up on FreeBSD within existing Windowsey infrastructure

Things you already know how to do:

* Firewall / router
* Free HTTPS: use `nginx` as a proxy, combine with `certbot` for **getting SSL certificates automatically and for free**
* DNS server (use NSD for authoritative or unbound for caching)

Things you almost know how to do:

* NextCloud server (like SharePoint, but it works! / like DropBox, but on your own infrastructure): `pkg install nextcloud` :-)
* Web server: `nginx` is a good server, combine with e.g. python... or `php72` if you really have to

More complicated, but still easy to integrate:

* File server on ZFS: use Samba to expose a ZFS filesystem to Windows machines
* spam filter: pass your email through postfix+rspamd and forward to your existing Exchange server

## More things to learn

All of this is in the handbook.

* Jails:
    * very useful lightweight isolation mechanism
    * like virtual machines, but more lightweight (less memory, less disk, faster)
    * if you have multiple independent things in the server, just make jails for them
    * see the handbook!
