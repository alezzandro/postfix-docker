# postfix-docker
Git repo for automated build of a postfix based docker container ready for OpenShift v3 (OSEv3)

This repository is used for sharing the work I've done for creating a postfix mailserver container.
I've made this container keeping in mind the basic requirements for running this container in any containers' management system.

# Environment variables
The docker container expects only one single env variable:
    *myhostname* (string) the Hostname on which postfix is exposed.

# TL;DR Description
I've successfully tested the container in various OpenShift environments, mainly as SMTP server (or with proper customization to the config file, as SMTP Relay server).

The most challenging part was to correctly place the logs for sent emails (by postfix) directly on STDOUT. 
Logging to STDOUT ensures that some logs grabber (Fluentd in OpenShift v3) could take these logs and send them to the default logs management system (EFK - Elasticsearch/Fluentd/Kibana on OSEv3).

Unfortunately postfix by default wants to send logs to some rsyslog, that usually saves them to a log file, so I've been forced to create postfix container using multiple processes, violating what should be the best practice for creating a container: 1 process per container.

The three processes involved in the container creation were:
- Postfix
- Rsyslog
- tail -f (on /var/log/maillog)

The last process to run and to leave in foreground, it was (for sure) the tail one. tail process will keep running in foreground while postfix and rsyslog do their job, but, what happen if postfix or rsyslog exits unexpectedly?
<br>*Nothing!*<br>
Docker container will keep running as soon as the tail process does. So your container orchestrator will never notice the absence of postfix process (unless you have some custom health check).
<br><br>
The solution I found was to use an internal processes orchestrator: supervisord (available in EPEL repo).
Supervisord will be responsible to watch and keep them running your configured processes. If a process exits unexpectedly it will run the process again.<br>
So your Docker's CMD/ENTRYPOINT will be the supervisord command.<br>
<br>
Looking at 'resources' folder, you'll find the config file for rsyslog and supervisord, all the conf for postfix are done in place directly on Dockerfile.<br>
Please note: as I said before, the container is pre-configured as a standard SMTP server, if you need some other configuration (for example: auth, relay, etc.) you should make the changes by yourself editing the Dockerfile.<br>
<br>
As you can see in the root directory you'll find also the rhel7 version of the Dockerfile. Please keep in mind that for producing a working postfix container based on Red Hat Enterprise Linux 7 you should run the 'docker build' command on a RHEL7 machine.<br>
<br>
<br>
Cheers.

# References
This container is also used in a Gitlab template, useful to create a persisten Gitlab on OSEv3 platform.
More info at:
https://github.com/sdellang/gitlab-openshift-persistent
