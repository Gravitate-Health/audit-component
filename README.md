# Kubernetes KSI + Rsyslog log handling service

This project contains a log handling and signing service setup leveraging Rsyslog and Guardtime KSI. It contains a Docker file to rebuild the image for Kubernetes that runs the service.

## Rebuild Docker image
Move to root folder of the project.
```sh
docker build --no-cache -t gt/ksi-rsyslog-base image
```

## Configuration.
Before usage service has to be configured. At first KSI user info has to be provided as a secret. At next there are several configuration files that can be modified.


### configmaps/rsyslog.conf
Default `rsyslog.conf` should work but there are some key points that should be brought up:
 - KSI user info is extracted from environment variables defined by `ksi-user-info-secret`.
 - Log files are filtered and signed by syslog tag.
 - Default log directory is `/var/log-ksi`.
 - Default log file name is `srv-<syslog tag>.log`.
 - Default input is from TCP 10514.
 - When other inputs are needed corresponding rsyslog modules has to be loaded.

### configmaps/logrotate-rsyslog
Default `logrotate-rsyslog` should work but there are some key points that should be brought up:
 - Pattern `/var/log-ksi/srv-*.log` should be changed if default log output folder or file name is changed in `rsyslog.conf`.
 - Note that pattern is followed by `{`. There must not be a space between!
 - Read notes in `logrotate-rsyslog`
 - Log rotation automatically integrates log signature files (see man logksi-integrate).

### configmaps/logrotate-rsyslog-ksi-common.conf
Default `logrotate-rsyslog-ksi-common.conf` should work but there are some key points that should be brought up:
 - Option `olddir /var/log-ksi` should be changed if default log output folder or file name is changed in `rsyslog.conf` and rotated logs are held in the same directory.
 - Known Bug: when log directories are mounted, log rotation only works if performed in same directory with active log files!
 - By default logs are rotated daily and 3 rounds of rotation is kept (see man logrotate).

### Make secret and config map
 
Move to root folder of the project.
```sh
# Note:
# Note:
# Note:
# Note that ksi+http is mandatory URI scheme! ksi+https and ksi+tcp are the only alternatives.

kubectl create secret generic ksi-user-info-secret \
        --from-literal=ksi.aggr.url=ksi+http://test/gt-signingservice \
        --from-literal=ksi.aggr.user=my_user \
        --from-literal=ksi.aggr.key=my_key

kubectl create configmap ksi-rsyslog-conf \
        --from-file=configmaps/rsyslog.conf \
        --from-file=configmaps/logrotate-rsyslog \
        --from-file=configmaps/logrotate-rsyslog-ksi-common.conf
```

## Deployment.

To deploy the service see `ksi-rsyslog-deployment.yml` and `ksi-rsyslog-service.yml`

Default `ksi-rsyslog-deployment.yml` should work but there are some key points that should be brought up:
 - The log directory is mounted on clusters host `/var/log-ksi`. If log or log rotation path is changed this must also be changed.

Default `ksi-rsyslog-service.yml` should work but there are some key points that should be brought up:
 - Default service name is `ksi-rsyslog-service` and port `16514`.
 
Move to root folder of the project.
```sh
kubectl apply -f ksi-rsyslog-deployment.yml
kubectl apply -f ksi-rsyslog-service.yml
```

## About Log signatures
Rsyslog KSI module works in asynchronous mode that separates log signature aggregation from issuing KSI signatures. Log signature is constructed into two files that has to be integrated into final log signature. That can be done with `logksi integrate` command. For current service integration is done by log rotation (see `/etc/logrotate-rsyslog-ksi-integrate.conf` for mode details).

To examine how this service is working, enter into the POD.
```sh
kubectl get pods
# Copy pod full name and replace it in the command:
# kubectl exec --stdin --tty ksi-rsyslog-deployment-546985b494-h4h9r -- /bin/bash

# To create logs on the same POD
logger -n 127.0.0.1 -t own-log -P 10514 -T "My Message"
ls /var/log-ksi

# You should see:
# srv-own-log.log
# srv-own-log.log.logsig.parts

# Force rsyslog to close and sign log block:
# See if and what is in log files:
pkill -1 rsyslogd
gttlvdump -pP /var/log-ksi/srv-own-log.log.logsig.parts/blocks.dat | less
gttlvdump -pP /var/log-ksi/srv-own-log.log.logsig.parts/block-signatures.dat | less

# Force log rotation 
logrotate --force /etc/logrotate.d/logrotate-rsyslog
ls /var/log-ksi
# You should see:
# srv-own-log.log.1
# srv-own-log.log.1.logsig

# Verify log signature
logksi verify --ver-key -d -- /var/log-ksi/srv-own-log.log.2 /var/log-ksi/srv-own-log.log.1

# When repeating the process and rotating once more verification should be done
# as follows:

```

Accessing logging service from the cluster:
```sh
# If ksi-rsyslog-service does not work,
# fix DNS or use CLUSTER-IP (kubectl get service)
logger -n ksi-rsyslog-service -t minikube-debug -P 16514 -T "My Message"
```
