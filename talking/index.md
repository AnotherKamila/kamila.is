---
title: Talks
subtitle: Slides for some of the talks I gave in various places.
menu_order: 30
grid: 3

layout: default
---

Slides for some of the talks I gave in various places.

Also lives on GitHub: [anotherkamila/talks](https://github.com/anotherkamila/talks).

* **SCION**
  {% for post in site.categories.talking %}
  {% if post.categories contains 'scion' %}
  * [{{post.title}}]({{post.url}})  
    {{post.venue}}, {{post.date | date: "%Y/%m/%d"}}
  {% endif %}
  {% endfor %}
  * multiple presentations on my HW router project (newer are hopefully better):
    * [P4 in the wild: Line-rate packet forwarding for the SCION future Internet architecture](scion/p4-scion-sdn-ch)  
      SDN Switzerland workshop, 2019-07-05
    * [High-speed SCION border router with P4+NetFPGA](scion/p4-scion-sidn)  
      ETH / SIDN meeting, 2019-04-11
* **FreeBSD**
  * [FreeBSD for Linux Refugees](to-linux-refugees)
* **Project-related**
  * SDN / P4
    * ["Load-aware" L4 load balancing in P4](sdn-loadbalancing)
* **Security**
  * [Paper Overview: A Comprehensive Formal Security Analysis of OAuth 2.0](oauth)
