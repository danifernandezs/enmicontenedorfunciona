---
layout: post
title:  "Regenera o expande un PVC de Elasticsearch del stack de OpenShift"
author: danifernandezs
categories: [ ocp4, efk, elasticsearch, pvc, storage, issue ]
image: assets/images/posts/2025-06-01/efk_stack_expandir_regenerar_pvc.jpg
featured: false
hidden: false
toc: false
---

Es posible que en producción te hayas quedado sin espacio en disco para tu `Elasticsearch`, o incluso que hayas perdido/borrado el volumen persistente.<br>
No te preocupes: aquí vamos a ver las dos soluciones, tanto para regenerar el `PVC de Elasticsearch` como para expandirlo, de modo que puedas seguir trabajando sin problemas.

# Redimensionando un PVC de Elasticsearch

Vamos a comenzar por la opción de `redimensionar` el PVC de Elasticsearch ya que seguramente nos hemos quedado sin espacio y no queremos perder nuevos logs que se estén generando en la plataforma.

Nos daremos cuenta de esta situación tanto por métricas de disco que monitorizamos como por la alerta por defecto que incluye el `stack EFK de OpenShift`.

La alerta se verá así:
```bash
Disk High Watermark Reached at Pod <elasticsearch_pod_name>.
Some shards will be re-allocated to different nodes if possible.
Make sure more disk space is added to the node or drop old indices allocated to this node.
```

Comprobamos el estado de uso de disco en `Elasticsearch`, usando la opción de visualización de Métricas que incluye OpenShift.
```bash
sum by (instance, pod) (round((1 - ( es_fs_path_available_bytes / es_fs_path_total_bytes )) * 100 , 0.0001))
```

En este caso, recibimos un resultado tal que así:
```bash
instance                  pod                                                 Value
10.128.2.34:60001         elasticsearch-cdm-cppltvjo-3-5895f4fb67-6cv5z       82.3831
10.129.3.32:60001         elasticsearch-cdm-cppltvjo-1-fdf9d79b5-j77tn        82.3809
10.131.0.144:60001        elasticsearch-cdm-cppltvjo-2-5998586f48-wz8xh       82.3796
```

Claramente nos estamos acercando al límite de uso de disco, por lo que debemos realizar el redimensionamiento para tener más espacio y seguir almacenando logs.

## Vamos a ello

El resumen de los pasos que vamos a seguir son:
- Comprobar el estado del clúster de Elasticsearch
- Listaremos los deployments de Elasticsearch
- Comprobaremos los PVCs de Elasticsearch
- Editamos el recurso ClusterLogging para redimensionar el PVC de Elasticsearch
- Forzamos la recreación del PVC de Elasticsearch

### Estado del clúster de Elasticsearch

Buscamos que el clúster esté en estado sano (“green”) y que no tengamos `unassigned shards`.
```bash
$ oc get clusterlogging instance -n openshift-logging -ojson | jq .status.logStore.elasticsearchStatus[].cluster

{
  "activePrimaryShards": 12,
  "activeShards": 24,
  "initializingShards": 0,
  "numDataNodes": 3,
  "numNodes": 3,
  "pendingTasks": 0,
  "relocatingShards": 0,
  "status": "green",            <------------
  "unassignedShards": 0         <------------
}
```

### Listando los deployments de Elasticsearch

Vamos a listar los `deployments` de Elasticsearch y los `pods` que están asociados a ellos.
```bash
$ oc get deployments -l component=elasticsearch -n openshift-logging

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
elasticsearch-cdm-cppltvjo-1   1/1     1            1           55m
elasticsearch-cdm-cppltvjo-2   1/1     1            1           55m
elasticsearch-cdm-cppltvjo-3   1/1     1            1           55m
```
```bash
$ oc get pods -n openshift-logging -l component=elasticsearch

NAME                                            READY   STATUS    RESTARTS   AGE
elasticsearch-cdm-cppltvjo-1-fdf9d79b5-j77tn    2/2     Running   0          55m
elasticsearch-cdm-cppltvjo-2-5998586f48-wz8xh   2/2     Running   0          55m
elasticsearch-cdm-cppltvjo-3-5895f4fb67-6cv5z   2/2     Running   0          55m
```

### Comprobando los PVCs de Elasticsearch

Listamos los `PVCs` de los `deployments`.
```bash
$ oc get pvc -n openshift-logging

NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-elasticsearch-cdm-cppltvjo-1   Bound    pvc-16507bf4-bba0-408a-b856-4c85c32f6fe0   187Gi      RWO            gp3-csi        55m
elasticsearch-elasticsearch-cdm-cppltvjo-2   Bound    pvc-3fa6a072-e475-4e5b-a62f-845d586277c5   187Gi      RWO            gp3-csi        55m
elasticsearch-elasticsearch-cdm-cppltvjo-3   Bound    pvc-9059bfec-eb4d-4d76-9899-82920b6baa36   187Gi      RWO            gp3-csi        55m
```

### Editando el recurso ClusterLogging y redimensionando el PVC de Elasticsearch

Estamos listos para redimensionar el espacio solicitado y forzar la recreación del PVC de Elasticsearch.

Vamos a editar primero el recurso `ClusterLogging` para modificar el tamaño del PVC definido.
```bash
$ oc edit clusterlogging.logging instance -n openshift-logging
```

Editamos la siguiente sección:
```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogging

...

spec:
  logStore:
    elasticsearch:
      nodeCount: 3
      storage:
        size: 500Gi     <------ Hemos aumentado a 500Gi la definición
```

De esta forma estamos listos para `forzar la recreación del PVC` de Elasticsearch.<br> Para ello, escalaremos los deployments a 0, eliminando el PVC asociado a dicho deployment para luego volver a escalarlo a 1. <br>

Así conseguiremos que OpenShift solicite un nuevo PVC, en este caso con el nuevo tamaño configurado.<br>Luego deberemos esperar a que el `clúster de Elasticsearch` se recupere, replique los datos y vuelva a estar en estado sano ("green").

Escalamos a 0 uno de los deployments, por ejemplo:
````bash
$ oc get deployments -l component=elasticsearch -n openshift-logging

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
elasticsearch-cdm-cppltvjo-1   1/1     1            1           57m   <----------------
elasticsearch-cdm-cppltvjo-2   1/1     1            1           57m
elasticsearch-cdm-cppltvjo-3   1/1     1            1           57m
```

```bash
$ oc scale deployment elasticsearch-cdm-cppltvjo-1 -n openshift-logging --replicas=0

deployment.apps/elasticsearch-cdm-cppltvjo-1 scaled
```

```bash
$ oc get deployments -l component=elasticsearch -n openshift-logging

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
elasticsearch-cdm-cppltvjo-1   0/0     0            0           58m
elasticsearch-cdm-cppltvjo-2   1/1     1            1           58m
elasticsearch-cdm-cppltvjo-3   1/1     1            1           58m
```

```bash
$ oc get pvc -n openshift-logging

NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-elasticsearch-cdm-cppltvjo-1   Bound    pvc-16507bf4-bba0-408a-b856-4c85c32f6fe0   187Gi      RWO            gp3-csi        58m       <----------------
elasticsearch-elasticsearch-cdm-cppltvjo-2   Bound    pvc-3fa6a072-e475-4e5b-a62f-845d586277c5   187Gi      RWO            gp3-csi        58m
elasticsearch-elasticsearch-cdm-cppltvjo-3   Bound    pvc-9059bfec-eb4d-4d76-9899-82920b6baa36   187Gi      RWO            gp3-csi        58m
```

```bash
$ oc delete pvc elasticsearch-elasticsearch-cdm-cppltvjo-1 -n openshift-logging

persistentvolumeclaim "elasticsearch-elasticsearch-cdm-cppltvjo-1" deleted
```

Ya podemos volver a escalar el deployment a 1, lo que provocará que se cree un `nuevo PVC` con el tamaño solicitado y el pod de Elasticsearch se vuelva a crear.
```bash
$ oc get deployments -l component=elasticsearch -n openshift-logging

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
elasticsearch-cdm-cppltvjo-1   0/0     0            0           61m
elasticsearch-cdm-cppltvjo-2   1/1     1            1           61m
elasticsearch-cdm-cppltvjo-3   1/1     1            1           61m
```

```bash
$ oc scale deployment elasticsearch-cdm-cppltvjo-1 -n openshift-logging --replicas=1

deployment.apps/elasticsearch-cdm-cppltvjo-1 scaled
```

```bash
$ oc get deployments -l component=elasticsearch -n openshift-logging

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
elasticsearch-cdm-cppltvjo-1   0/1     1            0           63m
elasticsearch-cdm-cppltvjo-2   1/1     1            1           63m
elasticsearch-cdm-cppltvjo-3   1/1     1            1           63m
```

```bash
$ oc get pods -n openshift-logging -l component=elasticsearch

NAME                                            READY   STATUS              RESTARTS   AGE
elasticsearch-cdm-cppltvjo-1-fdf9d79b5-qh8mq    0/2     ContainerCreating   0          10s
elasticsearch-cdm-cppltvjo-2-5998586f48-wz8xh   2/2     Running             0          63m
elasticsearch-cdm-cppltvjo-3-5895f4fb67-6cv5z   2/2     Running             0          63m
```

```bash
$ oc get pvc -n openshift-logging
NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-elasticsearch-cdm-cppltvjo-1   Bound    pvc-b0664be0-efc3-4976-9059-e79598bd47bc   500Gi      RWO            gp3-csi        14s
elasticsearch-elasticsearch-cdm-cppltvjo-2   Bound    pvc-3fa6a072-e475-4e5b-a62f-845d586277c5   187Gi      RWO            gp3-csi        63m
elasticsearch-elasticsearch-cdm-cppltvjo-3   Bound    pvc-9059bfec-eb4d-4d76-9899-82920b6baa36   187Gi      RWO            gp3-csi        63m
```

Comprobamos el estado del clúster (en “yellow” durante la replicación):
```bash
$ oc get clusterlogging instance -n openshift-logging -ojson | jq .status.logStore.elasticsearchStatus[].cluster

{
  "activePrimaryShards": 12,
  "activeShards": 24,
  "initializingShards": 0,
  "numDataNodes": 3,
  "numNodes": 3,
  "pendingTasks": 0,
  "relocatingShards": 0,
  "status": "yellow",            <------------
  "unassignedShards": 4          <------------
}
```

Comprobamos el estado de replicación, este proceso tardará tiempo y no debemos interrumpirlo. Podemos mantener un watch para comprobar el avance y estado del proceso.
```bash
$ watch oc exec -c elasticsearch elasticsearch-cdm-cppltvjo-1-fdf9d79b5-qh8mq -n openshift-logging -- es_util --query=_cat/health?v

Cada 2,0s: oc exec -c elasticsearch elasticsearch-cdm-cppltvjo-1-fdf9d79b5-qh8mq -n openshift-logging -- es_util --query=_cat/health?v

epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1748767741 08:49:01  elasticsearch yellow          3         3     24  12    0    0       4              0                  -                 64.9%
```

Tras un tiempo, el clúster ya habrá replicado los datos y estará en estado sano ("green") de nuevo.
```bash
Cada 2,0s: oc exec -c elasticsearch elasticsearch-cdm-cppltvjo-1-fdf9d79b5-qh8mq -n openshift-logging -- es_util --query=_cat/health?v

epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1748772981 10:16:21  elasticsearch green           3         3     24  12    0    0        0             0                  -                100.0%
```

```bash
$ oc get clusterlogging instance -n openshift-logging -ojson | jq .status.logStore.elasticsearchStatus[].cluster

{
  "activePrimaryShards": 12,
  "activeShards": 24,
  "initializingShards": 0,
  "numDataNodes": 3,
  "numNodes": 3,
  "pendingTasks": 0,
  "relocatingShards": 0,
  "status": "green",
  "unassignedShards": 0
}
```

A partir de este momento `repetiremos los pasos` para el resto de deployments de Elasticsearch, escalando a 0, eliminando el PVC y volviendo a escalar a 1.

Una vez que hayamos repetido el proceso para todos los deployments, ya tendremos el `PVC` de Elasticsearch redimensionado y funcionando correctamente.

```bash
$ oc get deployments -l component=elasticsearch -n openshift-logging

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
elasticsearch-cdm-cppltvjo-1   1/1     1            1           86m
elasticsearch-cdm-cppltvjo-2   1/1     1            1           86m
elasticsearch-cdm-cppltvjo-3   1/1     1            1           86m
```

```bash
$ oc get pods -n openshift-logging -l component=elasticsearch

NAME                                            READY   STATUS    RESTARTS   AGE
elasticsearch-cdm-cppltvjo-1-fdf9d79b5-qh8mq    2/2     Running   0          22m
elasticsearch-cdm-cppltvjo-2-5998586f48-zgjzh   2/2     Running   0          13m
elasticsearch-cdm-cppltvjo-3-5895f4fb67-ghb8p   2/2     Running   0          2m32s
```

```bash
$ oc get pvc -n openshift-logging

NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-elasticsearch-cdm-cppltvjo-1   Bound    pvc-b0664be0-efc3-4976-9059-e79598bd47bc   500Gi      RWO            gp3-csi        28m
elasticsearch-elasticsearch-cdm-cppltvjo-2   Bound    pvc-3c8a98a8-f696-4217-b63b-0a294008d545   500Gi      RWO            gp3-csi        12m
elasticsearch-elasticsearch-cdm-cppltvjo-3   Bound    pvc-020418bf-bc0c-4185-9e08-f11f2650d009   500Gi      RWO            gp3-csi        3m1s
```

```bash
$ oc exec -c elasticsearch elasticsearch-cdm-cppltvjo-1-fdf9d79b5-qh8mq -n openshift-logging -- es_util --query=_cat/health?v
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1748774320 10:38:40  elasticsearch green           3         3     24  12    0    0        0             0                  -                100.0%
```

```bash
$ oc get clusterlogging instance -n openshift-logging -ojson | jq .status.logStore.elasticsearchStatus[].cluster

{
  "activePrimaryShards": 12,
  "activeShards": 24,
  "initializingShards": 0,
  "numDataNodes": 3,
  "numNodes": 3,
  "pendingTasks": 0,
  "relocatingShards": 0,
  "status": "green",
  "unassignedShards": 0
}
```

# Recreación de un disco de Elasticsearch

Aunque es raro, puede suceder que un volumen de almacenamiento se elimine accidentalmente desde el hiperescalador. Cuando esto ocurra, uno de los pods de Elasticsearch quedará en estado `ContainerCreating` y el clúster pasará a `yellow` o `red`:

```bash
$ oc get pods -n openshift-logging -l component=elasticsearch

NAME                                            READY   STATUS              RESTARTS   AGE
elasticsearch-cdm-cppltvjo-1-fdf9d79b5-qh8mq    2/2     Running             0          5h40m
elasticsearch-cdm-cppltvjo-2-5998586f48-jbpz7   0/2     ContainerCreating   0          5m16s
elasticsearch-cdm-cppltvjo-3-5895f4fb67-ksx9k   2/2     Running             0          5h13m
```

```bash
$ oc get clusterlogging instance -n openshift-logging -ojson | jq .status.logStore.elasticsearchStatus[].cluster

{
  "activePrimaryShards": 12,
  "activeShards": 19,
  "initializingShards": 4,
  "numDataNodes": 3,
  "numNodes": 3,
  "pendingTasks": 1,
  "relocatingShards": 0,
  "status": "yellow",
  "unassignedShards": 1
}
```

Al revisar eventos, veremos el error de montaje del PV porque el volumen no existe:
```bash
$ oc get events -n openshift-logging --sort-by='.lastTimestamp'

...

119s        Warning   FailedAttachVolume     pod/elasticsearch-cdm-cppltvjo-2-5998586f48-jbpz7    (combined from similar events): AttachVolume.Attach failed for volume "pvc-3c8a98a8-f696-4217-b63b-0a294008d545" : rpc error: code = Internal desc = Could not get volume with ID "vol-0a465bba35f53d1d0": InvalidVolume.NotFound: The volume 'vol-0a465bba35f53d1d0' does not exist....

...
```

Podemos ver que el volumen en AWS no está presente, por lo que nuestra solución pasa por realizar unos pasos muy similares a los que hemos visto en el caso de redimensionar el PVC de Elasticsearch, es decir:
- Escalar a 0 el deployment de Elasticsearch
- Eliminar el PV y PVC asociados a ese volumen
- Reescalar el deployment de Elasticsearch a 1, lo que provocará que se cree un nuevo PVC y PV con el tamaño solicitado

Primero, identificamos el PV con el volumen perdido:
```bash
$ oc get pv

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                          STORAGECLASS   REASON   AGE
pvc-020418bf-bc0c-4185-9e08-f11f2650d009   500Gi      RWO            Delete           Bound    openshift-logging/elasticsearch-elasticsearch-cdm-cppltvjo-3   gp3-csi                 5h25m
pvc-3c8a98a8-f696-4217-b63b-0a294008d545   500Gi      RWO            Delete           Bound    openshift-logging/elasticsearch-elasticsearch-cdm-cppltvjo-2   gp3-csi                 5h35m
pvc-b0664be0-efc3-4976-9059-e79598bd47bc   500Gi      RWO            Delete           Bound    openshift-logging/elasticsearch-elasticsearch-cdm-cppltvjo-1   gp3-csi                 5h45m
```

El campo que nos interesa comprobar es `spec.csi.volumeHandle`, que es el que contiene el ID del volumen en el hiperescalador.

Nos coincide que el PV que presenta el error del volumen "perdido" es:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pvc-3c8a98a8-f696-4217-b63b-0a294008d545

...

spec:
  csi:
    volumeHandle: vol-0a465bba35f53d1d0

...

```

Y ese volumen corresponde al PVC:
```bash
$ oc get pvc

NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-elasticsearch-cdm-cppltvjo-1   Bound    pvc-b0664be0-efc3-4976-9059-e79598bd47bc   500Gi      RWO            gp3-csi        5h53m
elasticsearch-elasticsearch-cdm-cppltvjo-2   Bound    pvc-3c8a98a8-f696-4217-b63b-0a294008d545   500Gi      RWO            gp3-csi        5h37m         <----------------
elasticsearch-elasticsearch-cdm-cppltvjo-3   Bound    pvc-020418bf-bc0c-4185-9e08-f11f2650d009   500Gi      RWO            gp3-csi        5h27m
```

Por lo que realizaremos el procedimiento de escalado a 0 del deployment, eliminación del PVC y PV, y posterior escalado a 1 del deployment elasticsearch-cdm-cppltvjo-2 para forzar la creación de un nuevo PVC y PV.
```bash
$ oc get deployments -l component=elasticsearch -n openshift-logging

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
elasticsearch-cdm-cppltvjo-1   1/1     1            1           6h52m
elasticsearch-cdm-cppltvjo-2   0/1     1            0           6h52m
elasticsearch-cdm-cppltvjo-3   1/1     1            1           6h52m
```

```bash
$ oc scale deployment elasticsearch-cdm-cppltvjo-2 -n openshift-logging --replicas=0

deployment.apps/elasticsearch-cdm-cppltvjo-2 scaled
```

```bash
$ oc get pv

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                          STORAGECLASS   REASON   AGE
pvc-020418bf-bc0c-4185-9e08-f11f2650d009   500Gi      RWO            Delete           Bound    openshift-logging/elasticsearch-elasticsearch-cdm-cppltvjo-3   gp3-csi                 5h29m
pvc-3c8a98a8-f696-4217-b63b-0a294008d545   500Gi      RWO            Delete           Bound    openshift-logging/elasticsearch-elasticsearch-cdm-cppltvjo-2   gp3-csi                 5h39m
pvc-b0664be0-efc3-4976-9059-e79598bd47bc   500Gi      RWO            Delete           Bound    openshift-logging/elasticsearch-elasticsearch-cdm-cppltvjo-1   gp3-csi                 5h49m
```

```bash
$ oc delete pv pvc-3c8a98a8-f696-4217-b63b-0a294008d545

persistentvolume "pvc-3c8a98a8-f696-4217-b63b-0a294008d545" deleted
```

```bash
$ oc get pvc -n openshift-logging

NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-elasticsearch-cdm-cppltvjo-1   Bound    pvc-b0664be0-efc3-4976-9059-e79598bd47bc   500Gi      RWO            gp3-csi        5h54m
elasticsearch-elasticsearch-cdm-cppltvjo-2   Bound    pvc-3c8a98a8-f696-4217-b63b-0a294008d545   500Gi      RWO            gp3-csi        5h39m
elasticsearch-elasticsearch-cdm-cppltvjo-3   Bound    pvc-020418bf-bc0c-4185-9e08-f11f2650d009   500Gi      RWO            gp3-csi        5h29m
```

```bash
$ oc delete pvc elasticsearch-elasticsearch-cdm-cppltvjo-2 -n openshift-logging

persistentvolumeclaim "elasticsearch-elasticsearch-cdm-cppltvjo-2" deleted
```

```bash
$ oc get pvc -n openshift-logging

NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-elasticsearch-cdm-cppltvjo-1   Bound    pvc-b0664be0-efc3-4976-9059-e79598bd47bc   500Gi      RWO            gp3-csi        5h54m
elasticsearch-elasticsearch-cdm-cppltvjo-3   Bound    pvc-020418bf-bc0c-4185-9e08-f11f2650d009   500Gi      RWO            gp3-csi        5h29m
```

```bash
$ oc get pv

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                          STORAGECLASS   REASON   AGE
pvc-020418bf-bc0c-4185-9e08-f11f2650d009   500Gi      RWO            Delete           Bound    openshift-logging/elasticsearch-elasticsearch-cdm-cppltvjo-3   gp3-csi                 5h29m
pvc-b0664be0-efc3-4976-9059-e79598bd47bc   500Gi      RWO            Delete           Bound    openshift-logging/elasticsearch-elasticsearch-cdm-cppltvjo-1   gp3-csi                 5h49m
```

Reescalamos el despliegue para que el volumen sea recreado, comprobamos la replicación de datos y el estado del clúster vuele a ser estable.
```bash
$ oc scale deployment elasticsearch-cdm-cppltvjo-2 -n openshift-logging --replicas=1

deployment.apps/elasticsearch-cdm-cppltvjo-2 scaled
```

```bash
$ oc get deployments -l component=elasticsearch -n openshift-logging

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
elasticsearch-cdm-cppltvjo-1   1/1     1            1           6h53m
elasticsearch-cdm-cppltvjo-2   0/1     1            0           6h53m
elasticsearch-cdm-cppltvjo-3   1/1     1            1           6h53m
```

```bash
$ oc get pods -n openshift-logging -l component=elasticsearch

NAME                                            READY   STATUS              RESTARTS   AGE
elasticsearch-cdm-cppltvjo-1-fdf9d79b5-qh8mq    2/2     Running             0          5h50m
elasticsearch-cdm-cppltvjo-2-5998586f48-b5ssl   0/2     ContainerCreating   0          11s
elasticsearch-cdm-cppltvjo-3-5895f4fb67-ksx9k   2/2     Running             0          5h24m
```

```bash
$ oc get pvc -n openshift-logging

NAME                                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-elasticsearch-cdm-cppltvjo-1   Bound    pvc-b0664be0-efc3-4976-9059-e79598bd47bc   500Gi      RWO            gp3-csi        5h55m
elasticsearch-elasticsearch-cdm-cppltvjo-2   Bound    pvc-bfd13bc9-b36e-4bfc-8ca9-b13a50f25ec4   500Gi      RWO            gp3-csi        35s   <----------------
elasticsearch-elasticsearch-cdm-cppltvjo-3   Bound    pvc-020418bf-bc0c-4185-9e08-f11f2650d009   500Gi      RWO            gp3-csi        5h30m
```

```bash
$ oc get pv

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                          STORAGECLASS   REASON   AGE
pvc-020418bf-bc0c-4185-9e08-f11f2650d009   500Gi      RWO            Delete           Bound    openshift-logging/elasticsearch-elasticsearch-cdm-cppltvjo-3   gp3-csi                 5h30m
pvc-b0664be0-efc3-4976-9059-e79598bd47bc   500Gi      RWO            Delete           Bound    openshift-logging/elasticsearch-elasticsearch-cdm-cppltvjo-1   gp3-csi                 5h50m
pvc-bfd13bc9-b36e-4bfc-8ca9-b13a50f25ec4   500Gi      RWO            Delete           Bound    openshift-logging/elasticsearch-elasticsearch-cdm-cppltvjo-2   gp3-csi                 19s     <----------------
```

```bash
$ oc exec -c elasticsearch elasticsearch-cdm-cppltvjo-2-5998586f48-b5ssl -n openshift-logging -- es_util --query=_cat/health?v

epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1748767741 08:49:01  elasticsearch yellow          3         3     24  12    0    0       4              0                  -                 64.9%
```

```bash
$ oc exec -c elasticsearch elasticsearch-cdm-cppltvjo-1-fdf9d79b5-qh8mq -n openshift-logging -- es_util --query=_cat/health?v
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1748793998 16:06:38  elasticsearch green           3         3     24  12    2    0        0             0                  -                100.0%
```

```bash
$ oc get clusterlogging instance -n openshift-logging -ojson | jq .status.logStore.elasticsearchStatus[].cluster

{
  "activePrimaryShards": 12,
  "activeShards": 24,
  "initializingShards": 0,
  "numDataNodes": 3,
  "numNodes": 3,
  "pendingTasks": 0,
  "relocatingShards": 2,
  "status": "green",
  "unassignedShards": 0
}
```

De esta forma hemos conseguido recrear el disco de Elasticsearch y que el clúster vuelva a estar en estado sano ("green").

# Referencias

<a href="https://access.redhat.com/solutions/6075191" target="_blank">Resize ElasticSearch PersistentVolumeClaim in RHOCP4
</a><br>
