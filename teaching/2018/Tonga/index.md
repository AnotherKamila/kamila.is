---
title: FreeBSD workshop for TongaCERT (2018)
---

# Resources

* FreeBSD Handbook: [PDF](https://download.freebsd.org/ftp/doc/en/books/handbook/book.pdf), [online](https://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/), or [EPUB](https://download.freebsd.org/ftp/doc/en/books/handbook/book.epub)
* FreeBSD downloads page: https://www.freebsd.org/where.html
* programs:
    * SSH client for Windows: [PuTTY](https://putty.org/)
* specialised FreeBSD-based systems
    * [pfSense](https://www.pfsense.org/): FreeBSD + PF-based firewall with a web interface and extra features out of the box
    * [FreeNAS](http://www.freenas.org/): pre-configured ZFS-based NAS
        * does not really add that much to plain FreeBSD -- especially because you are now all experts on ZFS!
    * [TrueOS](https://www.trueos.org/): FreeBSD with a graphical interface pre-configured with some nice features
        * you can get the same with just plain FreeBSD, but with TrueOS the nice extras are there right after installation
        
# Notes

* Playing with virtual networks in VirtualBox:
  * use a bridged adapter if you want the VM to get an IP address from the router (like your laptop does)
  * use "internal network" to connect VMs to one another (but not to the outside)
