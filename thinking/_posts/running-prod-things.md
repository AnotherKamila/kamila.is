---
title: Things you should know when running something in production
---

* monitoring
* automation
* incident management

------------

notes to self:

# What to teach

* Monitoring
  * alerts: actionable, and only the things that matter
  * monitoring config should not scale linearly with network size :D (i.e. if I add a machine, I don't need to change anything)
* Automation
  * the main point isn't saving time, the main point is consistency / automation is the only actually up to date documentation you can have
    * and that's why changing just this one thing by hand is bad: now your only documentation is inconsistent
* Incident management
  * poking person != high-level person != communication person
