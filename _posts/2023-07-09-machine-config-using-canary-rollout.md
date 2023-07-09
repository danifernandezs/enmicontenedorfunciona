---
layout: post
title:  "Aplicando Machine Configs a nuestros nodos usando Canary Rollout como estrategia"
author: danifernandezs
categories: [ ocp4, node, mco, machine-api]
image: assets/images/posts/2023-07-09/NodeCanaryRollOut.png
featured: false
hidden: false
toc: false
---

En algunos escenarios, puede que necesitemos tener un mejor control al aplicar configuraciones en los nodos de cómputo, por eso, podemos basarnos en la metodología de Canary Rollout del software y aplicarlo a los nodos de nuestros clústeres. <br><br>En aquellos escenarios en los que no podamos permitirnos probar configuraciones, esperar un reinicio y en el momento de comprobar la configuración, esta sea errónea, para situaciones como esta es donde nos podemos plantear esta metodología.

# Entorno de pruebas y pasos a alto nivel

- OpenShift Container Platform 4.10.54
- Creamos un nuevo Machine Config Pool temporal
- Etiquetamos un nodo, sobre el cual probaremos las configuraciones
- Una vez que las configuraciones han sido aplicadas, comprobamos que son correctas
- Etiquetamos el resto de nodos para incluirlos al pool temporal
- Una vez que todos los nodos ya tienen las configuraciones aplicadas, aplicamos las configuraciones al pool original
- Devolvemos los nodos a su pool original
- Eliminamos todas las configuraciones temporales
- Eliminamos el Machine Config Pool temporal

# Fundamentos para la prueba

El motivo por el cual esta metodología funciona es por la forma en la que se generan los renderizados de las configuraciones a aplicar a los nodos.

El Machine Config Operator genera los renders con la siguiente metodología:<br>
Los nombres se generan como: `rendered-worker-${HASH}` siendo el `${HASH}` basado en el contenido del Machine Config a aplicar. Por este motivo, como el Machine Config que se renderiza para el Machine Config Pool nuevo que usamos para los test tiene el mismo contenido que el que se empleará en el pool original, los nodos ya disponen de esa configuración aplicada y no sufren una reconfiguración adicional.

# Configuraciones que aplicaremos para la prueba

Dos configuraciones muy típicas, un ajuste en los servidores NTP a usar en los nodos y crear un fichero con contenido dentro del nodo. <br>Estas configuraciones las haremos siguiendo la documentación oficial, es decir, definiendo la configuración y generando el Machine Config por medio de Butane.

<a href="https://docs.openshift.com/container-platform/4.10/installing/installing_bare_metal_ipi/ipi-install-post-installation-configuration.html#configuring-ntp-for-disconnected-clusters_ipi-install-post-installation-configuration" target="_blank">Configuring NTP for disconnected clusters</a>

# Estado inicial del clúster

```bash
$ oc get clusterversion

NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.10.54   True        False         24h     Cluster version is 4.10.54



$ oc get mcp

NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-a19dd72d532309981e2bf7ab579fb8d4   True      False      False      3              3                   3                     0                      24h
worker   rendered-worker-6bcab86604a96434bc10ce9b575cbb73   True      False      False      3              3                   3                     0                      24h

$ oc get no

NAME                                         STATUS   ROLES    AGE   VERSION
ip-10-0-136-152.eu-west-1.compute.internal   Ready    master   24h   v1.23.12+8a6bfe4
ip-10-0-137-204.eu-west-1.compute.internal   Ready    worker   24h   v1.23.12+8a6bfe4
ip-10-0-178-143.eu-west-1.compute.internal   Ready    master   24h   v1.23.12+8a6bfe4
ip-10-0-191-211.eu-west-1.compute.internal   Ready    worker   24h   v1.23.12+8a6bfe4
ip-10-0-196-76.eu-west-1.compute.internal    Ready    worker   24h   v1.23.12+8a6bfe4
ip-10-0-216-112.eu-west-1.compute.internal   Ready    master   24h   v1.23.12+8a6bfe4
```
```bash
$ oc debug no/ip-10-0-137-204.eu-west-1.compute.internal -- bash -c 'chroot /host cat /etc/chrony.conf'
Starting pod/ip-10-0-137-204eu-west-1computeinternal-debug ...
To use host binaries, run `chroot /host`
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
pool 2.rhel.pool.ntp.org iburst

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync

# Enable hardware timestamping on all interfaces that support it.
#hwtimestamp *

# Increase the minimum number of selectable sources required to adjust
# the system clock.
#minsources 2

# Allow NTP client access from local network.
#allow 192.168.0.0/16

# Serve time even if not synchronized to a time source.
#local stratum 10

# Specify file containing keys for NTP authentication.
keyfile /etc/chrony.keys

# Get TAI-UTC offset and leap seconds from the system tz database.
leapsectz right/UTC

# Specify directory for log files.
logdir /var/log/chrony

# Select which information is logged.
#log measurements statistics tracking

Removing debug pod ...



$ oc debug no/ip-10-0-191-211.eu-west-1.compute.internal -- bash -c 'chroot /host cat /etc/chrony.conf'
Starting pod/ip-10-0-191-211eu-west-1computeinternal-debug ...
To use host binaries, run `chroot /host`
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
pool 2.rhel.pool.ntp.org iburst

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync

# Enable hardware timestamping on all interfaces that support it.
#hwtimestamp *

# Increase the minimum number of selectable sources required to adjust
# the system clock.
#minsources 2

# Allow NTP client access from local network.
#allow 192.168.0.0/16

# Serve time even if not synchronized to a time source.
#local stratum 10

# Specify file containing keys for NTP authentication.
keyfile /etc/chrony.keys

# Get TAI-UTC offset and leap seconds from the system tz database.
leapsectz right/UTC

# Specify directory for log files.
logdir /var/log/chrony

# Select which information is logged.
#log measurements statistics tracking

Removing debug pod ...



$ oc debug no/ip-10-0-196-76.eu-west-1.compute.internal -- bash -c 'chroot /host cat /etc/chrony.conf'
Starting pod/ip-10-0-196-76eu-west-1computeinternal-debug ...
To use host binaries, run `chroot /host`
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
pool 2.rhel.pool.ntp.org iburst

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync

# Enable hardware timestamping on all interfaces that support it.
#hwtimestamp *

# Increase the minimum number of selectable sources required to adjust
# the system clock.
#minsources 2

# Allow NTP client access from local network.
#allow 192.168.0.0/16

# Serve time even if not synchronized to a time source.
#local stratum 10

# Specify file containing keys for NTP authentication.
keyfile /etc/chrony.keys

# Get TAI-UTC offset and leap seconds from the system tz database.
leapsectz right/UTC

# Specify directory for log files.
logdir /var/log/chrony

# Select which information is logged.
#log measurements statistics tracking

Removing debug pod ...
```
```bash
$ oc debug no/ip-10-0-137-204.eu-west-1.compute.internal -- bash -c 'chroot /host cat /etc/test-file'
Starting pod/ip-10-0-137-204eu-west-1computeinternal-debug ...
To use host binaries, run `chroot /host`
cat: /etc/test-file: No such file or directory

Removing debug pod ...
error: non-zero exit code from debug container



$ oc debug no/ip-10-0-191-211.eu-west-1.compute.internal -- bash -c 'chroot /host cat /etc/test-file'
Starting pod/ip-10-0-191-211eu-west-1computeinternal-debug ...
To use host binaries, run `chroot /host`
cat: /etc/test-file: No such file or directory

Removing debug pod ...
error: non-zero exit code from debug container



$ oc debug no/ip-10-0-196-76.eu-west-1.compute.internal -- bash -c 'chroot /host cat /etc/test-file'
Starting pod/ip-10-0-196-76eu-west-1computeinternal-debug ...
To use host binaries, run `chroot /host`
cat: /etc/test-file: No such file or directory

Removing debug pod ...
error: non-zero exit code from debug container
```

# Creamos el Machine Config Pool temporal para aplicar las nuevas configuraciones

```yaml
---
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: worker-config-test
spec:
  machineConfigSelector:
    matchExpressions:
    - key: machineconfiguration.openshift.io/role
      operator: In
      values:
      - worker
      - testing
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/testing: ""
  paused: false
```
Aplicamos el Machine Config Pool y comprobamos su creación, estado y el Machine Config renderizado para ser aplicado a los nodos que sean agregados a este nuevo pool.
```bash
$ oc apply -f testing-mcp.yaml
machineconfigpool.machineconfiguration.openshift.io/worker-config-test created
```
```bash
$ oc get mcp

NAME                 CONFIG                                                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master               rendered-master-a19dd72d532309981e2bf7ab579fb8d4               True      False      False      3              3                   3                     0                      24h
worker               rendered-worker-6bcab86604a96434bc10ce9b575cbb73               True      False      False      3              3                   3                     0                      24h
worker-config-test   rendered-worker-config-test-6bcab86604a96434bc10ce9b575cbb73   True      False      False      0              0                   0                     0                      15s



$ oc get mc

NAME                                                           GENERATEDBYCONTROLLER                      IGNITIONVERSION   AGE
00-master                                                      a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
00-worker                                                      a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
01-master-container-runtime                                    a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
01-master-kubelet                                              a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
01-worker-container-runtime                                    a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
01-worker-kubelet                                              a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
99-master-generated-crio-seccomp-use-default                                                              3.2.0             24h
99-master-generated-registries                                 a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
99-master-ssh                                                                                             3.2.0             24h
99-worker-generated-crio-seccomp-use-default                                                              3.2.0             24h
99-worker-generated-registries                                 a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
99-worker-ssh                                                                                             3.2.0             24h
rendered-master-a19dd72d532309981e2bf7ab579fb8d4               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
rendered-worker-6bcab86604a96434bc10ce9b575cbb73               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
rendered-worker-8fc9ea9e12a96bad7ab0d89debbb5f3b               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             56m
rendered-worker-config-test-6bcab86604a96434bc10ce9b575cbb73   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             13s
rendered-worker-ff34160d1ad23679fd4d882c80a7ad68               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             56m
```

# Etiquetamos un nodo de cómputo sobre el que probaremos las configuraciones

```bash
$ oc label node ip-10-0-137-204.eu-west-1.compute.internal node-role.kubernetes.io/testing=
node/ip-10-0-137-204.eu-west-1.compute.internal labeled



$ oc get no

NAME                                         STATUS   ROLES            AGE   VERSION
ip-10-0-136-152.eu-west-1.compute.internal   Ready    master           24h   v1.23.12+8a6bfe4
ip-10-0-137-204.eu-west-1.compute.internal   Ready    testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-178-143.eu-west-1.compute.internal   Ready    master           24h   v1.23.12+8a6bfe4
ip-10-0-191-211.eu-west-1.compute.internal   Ready    worker           24h   v1.23.12+8a6bfe4
ip-10-0-196-76.eu-west-1.compute.internal    Ready    worker           24h   v1.23.12+8a6bfe4
ip-10-0-216-112.eu-west-1.compute.internal   Ready    master           24h   v1.23.12+8a6bfe4



$ oc get mcp

NAME                 CONFIG                                                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master               rendered-master-a19dd72d532309981e2bf7ab579fb8d4               True      False      False      3              3                   3                     0                      24h
worker               rendered-worker-6bcab86604a96434bc10ce9b575cbb73               True      False      False      2              2                   2                     0                      24h
worker-config-test   rendered-worker-config-test-6bcab86604a96434bc10ce9b575cbb73   True      False      False      1              1                   1                     0                      101s
```

# Creamos los Machine Configs y aplicamos

Ahora vamos a crear las configuraciones que queremos probar y que serán aplicadas al nodos de cómputo que hemos "movido" al nuevo Machine Config Pool.

Configuración para NTP
```yaml
variant: openshift
version: 4.10.0
metadata:
  name: 99-testing-worker-chrony
  labels:
    machineconfiguration.openshift.io/role: testing
storage:
  files:
  - path: /etc/chrony.conf
    mode: 0644 
    overwrite: true
    contents:
      inline: |
        pool time.google.com iburst 
        driftfile /var/lib/chrony/drift
        makestep 1.0 3
        rtcsync
        logdir /var/log/chrony
```

Fichero multilínea que agregaremos a los nodos
```yaml
variant: openshift
version: 4.10.0
metadata:
  name: 99-testing-worker-multilinefile
  labels:
    machineconfiguration.openshift.io/role: testing
storage:
  files:
  - path: /etc/test-file
    mode: 0644 
    overwrite: true
    contents:
      inline: |
        This is a multiline file
        with an aditional line
        and another new line
```

Generamos los Machine Configs

```bash
$ butane 99-testing-worker-chrony.bu -o 99-testing-worker-chrony.yaml
$ butane 99-testing-worker-multilinefile.bu -o 99-testing-worker-multilinefile.yaml
```
```yaml
$ cat 99-testing-worker-chrony.yaml 

# Generated by Butane; do not edit
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: testing
  name: 99-testing-worker-chrony
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - contents:
            compression: ""
            source: data:,pool%20time.google.com%20iburst%20%0Adriftfile%20%2Fvar%2Flib%2Fchrony%2Fdrift%0Amakestep%201.0%203%0Artcsync%0Alogdir%20%2Fvar%2Flog%2Fchrony%0A
          mode: 420
          overwrite: true
          path: /etc/chrony.conf
```
```yaml
$ cat 99-testing-worker-multilinefile.yaml 

# Generated by Butane; do not edit
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: testing
  name: 99-testing-worker-multilinefile
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - contents:
            compression: ""
            source: data:,This%20is%20a%20multiline%20file%0Awith%20an%20aditional%20line%0Aand%20another%20new%20line%0A
          mode: 420
          overwrite: true
          path: /etc/test-file
```

Aplicamos los recursos, comprobamos que ha sido renderizado el Machine Config y esperaremos a que las configuraciones sean aplicadas al nodo de cómputo.

```bash
$ oc apply -f 99-testing-worker-chrony.yaml
machineconfig.machineconfiguration.openshift.io/99-testing-worker-chrony created

$ oc apply -f 99-testing-worker-multilinefile.yaml
machineconfig.machineconfiguration.openshift.io/99-testing-worker-multilinefile created
```
```bash
$ oc get mc

NAME                                                           GENERATEDBYCONTROLLER                      IGNITIONVERSION   AGE
00-master                                                      a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
00-worker                                                      a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
01-master-container-runtime                                    a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
01-master-kubelet                                              a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
01-worker-container-runtime                                    a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
01-worker-kubelet                                              a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
99-master-generated-crio-seccomp-use-default                                                              3.2.0             24h
99-master-generated-registries                                 a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
99-master-ssh                                                                                             3.2.0             25h
99-testing-worker-chrony                                                                                  3.2.0             10s
99-testing-worker-multilinefile                                                                           3.2.0             4s
99-worker-generated-crio-seccomp-use-default                                                              3.2.0             24h
99-worker-generated-registries                                 a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
99-worker-ssh                                                                                             3.2.0             25h
rendered-master-a19dd72d532309981e2bf7ab579fb8d4               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
rendered-worker-6bcab86604a96434bc10ce9b575cbb73               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             24h
rendered-worker-8fc9ea9e12a96bad7ab0d89debbb5f3b               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             59m
rendered-worker-config-test-6bcab86604a96434bc10ce9b575cbb73   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             2m45s
rendered-worker-config-test-8fc9ea9e12a96bad7ab0d89debbb5f3b   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             5s
rendered-worker-ff34160d1ad23679fd4d882c80a7ad68               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             58m



$ oc get mcp

NAME                 CONFIG                                                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master               rendered-master-a19dd72d532309981e2bf7ab579fb8d4               True      False      False      3              3                   3                     0                      24h
worker               rendered-worker-6bcab86604a96434bc10ce9b575cbb73               True      False      False      2              2                   2                     0                      24h
worker-config-test   rendered-worker-config-test-6bcab86604a96434bc10ce9b575cbb73   False     True       False      1              0                   0                     0                      2m54s



$ oc get no

NAME                                         STATUS                     ROLES            AGE   VERSION
ip-10-0-136-152.eu-west-1.compute.internal   Ready                      master           25h   v1.23.12+8a6bfe4
ip-10-0-137-204.eu-west-1.compute.internal   Ready,SchedulingDisabled   testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-178-143.eu-west-1.compute.internal   Ready                      master           25h   v1.23.12+8a6bfe4
ip-10-0-191-211.eu-west-1.compute.internal   Ready                      worker           24h   v1.23.12+8a6bfe4
ip-10-0-196-76.eu-west-1.compute.internal    Ready                      worker           24h   v1.23.12+8a6bfe4
ip-10-0-216-112.eu-west-1.compute.internal   Ready                      master           25h   v1.23.12+8a6bfe4



$ oc get no

NAME                                         STATUS                        ROLES            AGE   VERSION
ip-10-0-136-152.eu-west-1.compute.internal   Ready                         master           25h   v1.23.12+8a6bfe4
ip-10-0-137-204.eu-west-1.compute.internal   NotReady,SchedulingDisabled   testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-178-143.eu-west-1.compute.internal   Ready                         master           25h   v1.23.12+8a6bfe4
ip-10-0-191-211.eu-west-1.compute.internal   Ready                         worker           24h   v1.23.12+8a6bfe4
ip-10-0-196-76.eu-west-1.compute.internal    Ready                         worker           24h   v1.23.12+8a6bfe4
ip-10-0-216-112.eu-west-1.compute.internal   Ready                         master           25h   v1.23.12+8a6bfe4



$ oc get no

NAME                                         STATUS   ROLES            AGE   VERSION
ip-10-0-136-152.eu-west-1.compute.internal   Ready    master           25h   v1.23.12+8a6bfe4
ip-10-0-137-204.eu-west-1.compute.internal   Ready    testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-178-143.eu-west-1.compute.internal   Ready    master           25h   v1.23.12+8a6bfe4
ip-10-0-191-211.eu-west-1.compute.internal   Ready    worker           24h   v1.23.12+8a6bfe4
ip-10-0-196-76.eu-west-1.compute.internal    Ready    worker           24h   v1.23.12+8a6bfe4
ip-10-0-216-112.eu-west-1.compute.internal   Ready    master           25h   v1.23.12+8a6bfe4
```

Revisamos que las configuraciones han sido aplicadas al nodo de cómputo.

```bash
$ oc debug no/ip-10-0-137-204.eu-west-1.compute.internal -- bash -c 'chroot /host cat /etc/chrony.conf'
Starting pod/ip-10-0-137-204eu-west-1computeinternal-debug ...
To use host binaries, run `chroot /host`
pool time.google.com iburst 
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony

Removing debug pod ...



$ oc debug no/ip-10-0-137-204.eu-west-1.compute.internal -- bash -c 'chroot /host cat /etc/test-file'
Starting pod/ip-10-0-137-204eu-west-1computeinternal-debug ...
To use host binaries, run `chroot /host`
This is a multiline file
with an aditional line
and another new line

Removing debug pod ...
```

Vemos que todas las configuraciones están aplicadas como deseábamos, por lo que han sido validadas.

# Movemos el resto de nodos

Movemos, etiquetando, el resto de nodos hacia el nuevo Machine Config Pool que hemos creado para que reciban las configuraciones y así luego poder aplicarlas al pool original y que los nodos no sufran un reinicio adicional.

```bash
$ oc label node ip-10-0-191-211.eu-west-1.compute.internal node-role.kubernetes.io/testing=
node/ip-10-0-191-211.eu-west-1.compute.internal labeled

$ oc label node ip-10-0-196-76.eu-west-1.compute.internal node-role.kubernetes.io/testing=
node/ip-10-0-196-76.eu-west-1.compute.internal labeled
```

```bash
$ oc get no

NAME                                         STATUS   ROLES            AGE   VERSION
ip-10-0-136-152.eu-west-1.compute.internal   Ready    master           25h   v1.23.12+8a6bfe4
ip-10-0-137-204.eu-west-1.compute.internal   Ready    testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-178-143.eu-west-1.compute.internal   Ready    master           25h   v1.23.12+8a6bfe4
ip-10-0-191-211.eu-west-1.compute.internal   Ready    testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-196-76.eu-west-1.compute.internal    Ready    testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-216-112.eu-west-1.compute.internal   Ready    master           25h   v1.23.12+8a6bfe4



$ oc get mcp

NAME                 CONFIG                                                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master               rendered-master-a19dd72d532309981e2bf7ab579fb8d4               True      False      False      3              3                   3                     0                      25h
worker               rendered-worker-6bcab86604a96434bc10ce9b575cbb73               True      False      False      0              0                   0                     0                      25h
worker-config-test   rendered-worker-config-test-ff34160d1ad23679fd4d882c80a7ad68   False     True       False      3              1                   1                     0                      5m44s



$ oc get no

NAME                                         STATUS                     ROLES            AGE   VERSION
ip-10-0-136-152.eu-west-1.compute.internal   Ready                      master           25h   v1.23.12+8a6bfe4
ip-10-0-137-204.eu-west-1.compute.internal   Ready                      testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-178-143.eu-west-1.compute.internal   Ready                      master           25h   v1.23.12+8a6bfe4
ip-10-0-191-211.eu-west-1.compute.internal   Ready,SchedulingDisabled   testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-196-76.eu-west-1.compute.internal    Ready                      testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-216-112.eu-west-1.compute.internal   Ready                      master           25h   v1.23.12+8a6bfe4



$ oc get no

NAME                                         STATUS                        ROLES            AGE   VERSION
ip-10-0-136-152.eu-west-1.compute.internal   Ready                         master           25h   v1.23.12+8a6bfe4
ip-10-0-137-204.eu-west-1.compute.internal   Ready                         testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-178-143.eu-west-1.compute.internal   Ready                         master           25h   v1.23.12+8a6bfe4
ip-10-0-191-211.eu-west-1.compute.internal   NotReady,SchedulingDisabled   testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-196-76.eu-west-1.compute.internal    Ready                         testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-216-112.eu-west-1.compute.internal   Ready                         master           25h   v1.23.12+8a6bfe4



$ oc get no

NAME                                         STATUS   ROLES            AGE   VERSION
ip-10-0-136-152.eu-west-1.compute.internal   Ready    master           25h   v1.23.12+8a6bfe4
ip-10-0-137-204.eu-west-1.compute.internal   Ready    testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-178-143.eu-west-1.compute.internal   Ready    master           25h   v1.23.12+8a6bfe4
ip-10-0-191-211.eu-west-1.compute.internal   Ready    testing,worker   25h   v1.23.12+8a6bfe4
ip-10-0-196-76.eu-west-1.compute.internal    Ready    testing,worker   24h   v1.23.12+8a6bfe4
ip-10-0-216-112.eu-west-1.compute.internal   Ready    master           25h   v1.23.12+8a6bfe4



$ oc get no

NAME                                         STATUS                     ROLES            AGE   VERSION
ip-10-0-136-152.eu-west-1.compute.internal   Ready                      master           25h   v1.23.12+8a6bfe4
ip-10-0-137-204.eu-west-1.compute.internal   Ready                      testing,worker   25h   v1.23.12+8a6bfe4
ip-10-0-178-143.eu-west-1.compute.internal   Ready                      master           25h   v1.23.12+8a6bfe4
ip-10-0-191-211.eu-west-1.compute.internal   Ready                      testing,worker   25h   v1.23.12+8a6bfe4
ip-10-0-196-76.eu-west-1.compute.internal    Ready,SchedulingDisabled   testing,worker   25h   v1.23.12+8a6bfe4
ip-10-0-216-112.eu-west-1.compute.internal   Ready                      master           25h   v1.23.12+8a6bfe4



$ oc get no

NAME                                         STATUS                        ROLES            AGE   VERSION
ip-10-0-136-152.eu-west-1.compute.internal   Ready                         master           25h   v1.23.12+8a6bfe4
ip-10-0-137-204.eu-west-1.compute.internal   Ready                         testing,worker   25h   v1.23.12+8a6bfe4
ip-10-0-178-143.eu-west-1.compute.internal   Ready                         master           25h   v1.23.12+8a6bfe4
ip-10-0-191-211.eu-west-1.compute.internal   Ready                         testing,worker   25h   v1.23.12+8a6bfe4
ip-10-0-196-76.eu-west-1.compute.internal    NotReady,SchedulingDisabled   testing,worker   25h   v1.23.12+8a6bfe4
ip-10-0-216-112.eu-west-1.compute.internal   Ready                         master           25h   v1.23.12+8a6bfe4



$ oc get no

NAME                                         STATUS   ROLES            AGE   VERSION
ip-10-0-136-152.eu-west-1.compute.internal   Ready    master           25h   v1.23.12+8a6bfe4
ip-10-0-137-204.eu-west-1.compute.internal   Ready    testing,worker   25h   v1.23.12+8a6bfe4
ip-10-0-178-143.eu-west-1.compute.internal   Ready    master           25h   v1.23.12+8a6bfe4
ip-10-0-191-211.eu-west-1.compute.internal   Ready    testing,worker   25h   v1.23.12+8a6bfe4
ip-10-0-196-76.eu-west-1.compute.internal    Ready    testing,worker   25h   v1.23.12+8a6bfe4
ip-10-0-216-112.eu-west-1.compute.internal   Ready    master           25h   v1.23.12+8a6bfe4
```

# Replicamos las configuraciones al pool de nodos original

Replicamos las configuraciones que hemos probado en el nuevo Machine Config Pool pero al pool original, para que se renderice la configuración final y podamos volver a mover los nodos a su pool inicial y eliminar los pasos intermedios.

```yaml
$ cat 99-worker-chrony.bu
variant: openshift
version: 4.10.0
metadata:
  name: 99-worker-chrony
  labels:
    machineconfiguration.openshift.io/role: worker
storage:
  files:
  - path: /etc/chrony.conf
    mode: 0644 
    overwrite: true
    contents:
      inline: |
        pool time.google.com iburst 
        driftfile /var/lib/chrony/drift
        makestep 1.0 3
        rtcsync
        logdir /var/log/chrony

$ butane 99-worker-chrony.bu -o 99-worker-chrony.yaml

$ cat 99-worker-chrony.yaml
# Generated by Butane; do not edit
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-worker-chrony
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - contents:
            compression: ""
            source: data:,pool%20time.google.com%20iburst%20%0Adriftfile%20%2Fvar%2Flib%2Fchrony%2Fdrift%0Amakestep%201.0%203%0Artcsync%0Alogdir%20%2Fvar%2Flog%2Fchrony%0A
          mode: 420
          overwrite: true
          path: /etc/chrony.conf
```

```yaml
$ cat 99-worker-multilinefile.bu
variant: openshift
version: 4.10.0
metadata:
  name: 99-worker-multilinefile
  labels:
    machineconfiguration.openshift.io/role: worker
storage:
  files:
  - path: /etc/test-file
    mode: 0644 
    overwrite: true
    contents:
      inline: |
        This is a multiline file
        with an aditional line
        and another new line

$ butane 99-worker-multilinefile.bu -o 99-worker-multilinefile.yaml

$ cat 99-worker-multilinefile.yaml
# Generated by Butane; do not edit
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-worker-multilinefile
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - contents:
            compression: ""
            source: data:,This%20is%20a%20multiline%20file%0Awith%20an%20aditional%20line%0Aand%20another%20new%20line%0A
          mode: 420
          overwrite: true
          path: /etc/test-file
```

Aplicamos los recursos, comprobamos que se han renderizado.

```bash
$ oc apply -f 99-worker-chrony.yaml
machineconfig.machineconfiguration.openshift.io/99-worker-chrony created

$ oc apply -f 99-worker-multilinefile.yaml
machineconfig.machineconfiguration.openshift.io/99-worker-multilinefile created



$ oc get mc

NAME                                                           GENERATEDBYCONTROLLER                      IGNITIONVERSION   AGE
00-master                                                      a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
00-worker                                                      a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
01-master-container-runtime                                    a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
01-master-kubelet                                              a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
01-worker-container-runtime                                    a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
01-worker-kubelet                                              a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
99-master-generated-crio-seccomp-use-default                                                              3.2.0             25h
99-master-generated-registries                                 a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
99-master-ssh                                                                                             3.2.0             25h
99-testing-worker-chrony                                                                                  3.2.0             11m
99-testing-worker-multilinefile                                                                           3.2.0             11m
99-worker-chrony                                                                                          3.2.0             4s
99-worker-generated-crio-seccomp-use-default                                                              3.2.0             25h
99-worker-generated-registries                                 a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
99-worker-multilinefile                                                                                   3.2.0             4s
99-worker-ssh                                                                                             3.2.0             25h
rendered-master-a19dd72d532309981e2bf7ab579fb8d4               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
rendered-worker-6bcab86604a96434bc10ce9b575cbb73               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
rendered-worker-8fc9ea9e12a96bad7ab0d89debbb5f3b               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             70m
rendered-worker-config-test-6bcab86604a96434bc10ce9b575cbb73   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             14m
rendered-worker-config-test-8fc9ea9e12a96bad7ab0d89debbb5f3b   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             11m
rendered-worker-config-test-ff34160d1ad23679fd4d882c80a7ad68   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             11m
rendered-worker-ff34160d1ad23679fd4d882c80a7ad68               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             70m
```

# Devolvemos los nodos a su pool original

Para esto, eliminamos la etiqueta que agregamos de forma adicional, comprobamos que los nodos de cómputo vuelven a su Machine Config Pool y que no reciben una reconfiguración.

```bash
$ oc label node ip-10-0-137-204.eu-west-1.compute.internal node-role.kubernetes.io/testing-
node/ip-10-0-137-204.eu-west-1.compute.internal unlabeled

$ oc label node ip-10-0-191-211.eu-west-1.compute.internal node-role.kubernetes.io/testing-
node/ip-10-0-191-211.eu-west-1.compute.internal unlabeled

$ oc label node ip-10-0-196-76.eu-west-1.compute.internal node-role.kubernetes.io/testing-
node/ip-10-0-196-76.eu-west-1.compute.internal unlabeled



$ oc get no

NAME                                         STATUS   ROLES    AGE   VERSION
ip-10-0-136-152.eu-west-1.compute.internal   Ready    master   25h   v1.23.12+8a6bfe4
ip-10-0-137-204.eu-west-1.compute.internal   Ready    worker   25h   v1.23.12+8a6bfe4
ip-10-0-178-143.eu-west-1.compute.internal   Ready    master   25h   v1.23.12+8a6bfe4
ip-10-0-191-211.eu-west-1.compute.internal   Ready    worker   25h   v1.23.12+8a6bfe4
ip-10-0-196-76.eu-west-1.compute.internal    Ready    worker   25h   v1.23.12+8a6bfe4
ip-10-0-216-112.eu-west-1.compute.internal   Ready    master   25h   v1.23.12+8a6bfe4



$ oc get mcp

NAME                 CONFIG                                                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master               rendered-master-a19dd72d532309981e2bf7ab579fb8d4               True      False      False      3              3                   3                     0                      25h
worker               rendered-worker-99c11c8e1d9b701a645c623042bfca21               False     True       False      3              0                   0                     0                      25h
worker-config-test   rendered-worker-config-test-ff34160d1ad23679fd4d882c80a7ad68   True      False      False      3              3                   3                     0                      15m



$ oc get mcp

NAME                 CONFIG                                                         UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master               rendered-master-a19dd72d532309981e2bf7ab579fb8d4               True      False      False      3              3                   3                     0                      25h
worker               rendered-worker-99c11c8e1d9b701a645c623042bfca21               True      False      False      3              3                   3                     0                      25h
worker-config-test   rendered-worker-config-test-ff34160d1ad23679fd4d882c80a7ad68   True      False      False      0              0                   0                     0                      15m
```

# Limpiamos todos los recursos usados temporalmente

Eliminamos las configuraciones que se aplicaban al pool temporal y también el Machine Config Pool.

```bash
$ oc delete -f 99-testing-worker-chrony.yaml 
machineconfig.machineconfiguration.openshift.io "99-testing-worker-chrony" deleted

$ oc delete -f 99-testing-worker-multilinefile.yaml 
machineconfig.machineconfiguration.openshift.io "99-testing-worker-multilinefile" deleted



$ oc get mc

NAME                                                           GENERATEDBYCONTROLLER                      IGNITIONVERSION   AGE
00-master                                                      a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
00-worker                                                      a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
01-master-container-runtime                                    a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
01-master-kubelet                                              a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
01-worker-container-runtime                                    a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
01-worker-kubelet                                              a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
99-master-generated-crio-seccomp-use-default                                                              3.2.0             25h
99-master-generated-registries                                 a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
99-master-ssh                                                                                             3.2.0             25h
99-worker-chrony                                                                                          3.2.0             3m25s
99-worker-generated-crio-seccomp-use-default                                                              3.2.0             25h
99-worker-generated-registries                                 a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
99-worker-multilinefile                                                                                   3.2.0             3m25s
99-worker-ssh                                                                                             3.2.0             25h
rendered-master-a19dd72d532309981e2bf7ab579fb8d4               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
rendered-worker-6bcab86604a96434bc10ce9b575cbb73               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
rendered-worker-8fc9ea9e12a96bad7ab0d89debbb5f3b               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             73m
rendered-worker-99c11c8e1d9b701a645c623042bfca21               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             3m20s
rendered-worker-config-test-6bcab86604a96434bc10ce9b575cbb73   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             17m
rendered-worker-config-test-8fc9ea9e12a96bad7ab0d89debbb5f3b   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             14m
rendered-worker-config-test-99c11c8e1d9b701a645c623042bfca21   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             6s
rendered-worker-config-test-ff34160d1ad23679fd4d882c80a7ad68   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             14m
rendered-worker-ff34160d1ad23679fd4d882c80a7ad68               a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             73m



$ oc delete -f testing-mcp.yaml 
machineconfigpool.machineconfiguration.openshift.io "worker-config-test" deleted



$ oc get mcp

NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-a19dd72d532309981e2bf7ab579fb8d4   True      False      False      3              3                   3                     0                      25h
worker   rendered-worker-99c11c8e1d9b701a645c623042bfca21   True      False      False      3              3                   3                     0                      25h



$ oc get mc

NAME                                               GENERATEDBYCONTROLLER                      IGNITIONVERSION   AGE
00-master                                          a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
00-worker                                          a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
01-master-container-runtime                        a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
01-master-kubelet                                  a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
01-worker-container-runtime                        a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
01-worker-kubelet                                  a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
99-master-generated-crio-seccomp-use-default                                                  3.2.0             25h
99-master-generated-registries                     a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
99-master-ssh                                                                                 3.2.0             25h
99-worker-chrony                                                                              3.2.0             3m48s
99-worker-generated-crio-seccomp-use-default                                                  3.2.0             25h
99-worker-generated-registries                     a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
99-worker-multilinefile                                                                       3.2.0             3m48s
99-worker-ssh                                                                                 3.2.0             25h
rendered-master-a19dd72d532309981e2bf7ab579fb8d4   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
rendered-worker-6bcab86604a96434bc10ce9b575cbb73   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             25h
rendered-worker-8fc9ea9e12a96bad7ab0d89debbb5f3b   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             74m
rendered-worker-99c11c8e1d9b701a645c623042bfca21   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             3m43s
rendered-worker-ff34160d1ad23679fd4d882c80a7ad68   a21b2b845994ebceabd4f9fca97b04fc0d90d5a2   3.2.0             74m
```
