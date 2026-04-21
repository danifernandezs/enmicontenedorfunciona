---
layout: post
title:  "Encriptando datos en ETCD, y ahora también con cifrado AES-GCM"
author: danifernandezs
categories: [ ocp4, etcd, encryption, operator]
image: assets/images/posts/2023-06-24/etcd-encryption.png
featured: false
hidden: false
toc: false
---

Recordemos que en un clúster de OpenShift/Kubernetes se utiliza *etcd* como almacenamiento para el estado del clúster, pero de forma genérica, todos los datos almacenados están en texto plano.<br><br>
Esto implica que cualquier usuario con acceso a *etcd* tendrá acceso a información sensible almacenada, como por ejemplo, los datos que tengamos en el clúster en forma de secretos y configmaps.

# Sobre el encriptado de los datos en etcd

Por defecto, los datos almacenados en *etcd* en un clúster de OpenShift no están encriptados. Pero se puede habilitar el encriptado para agregar una capa a mayores de seguridad a tu clúster.

Los datos que se encriptan una vez activado son:
- Secrets (Kubernetes API)
- Configmaps (Kubernetes API)
- Routes (OpenShift API)
- OAuth access tokens (OpenShift API)
- OAuth authorize tokens (OpenShift API)

## Importante

- Al activar el encriptado de *etcd*, se crean las claves de encriptación; estas claves son necesarias para restaurar un backup de etcd.
- Al realizar un backup de _etcd_, se crea un fichero *static_kuberesources_TIME.tar.gz* que contiene las claves de encriptado, por seguridad, recuerda almacenar este fichero separado del snapshot de _etcd_ que se haya realizado.
- Únicamente se encriptan los valores de los datos almacenados y no las claves, en el par key-value de *etcd*.
- Los nombres de los objetos no se encriptan.

# Tipos de encriptado que soporta OpenShift 4.13

<style>
  .tb {
    border-collapse: collapse;
    margin-left: auto;
    margin-right: auto;
  }
  .tb th, .tb td { padding: 5px; border: solid 1px #777; text-align: center;}
</style>

<table class="tb">
  <tr>
    <th>Nombre</th>
    <th>Encriptado</th>
    <th>Velocidad</th>
    <th>Rotación</th>
  </tr>
  <tr>
    <td>identity</td>
    <td>Sin encriptar</td>
    <td>N/A</td>
    <td>N/A</td>
  </tr>
  <tr>
    <td>aescbc</td>
    <td>AES-CBC</td>
    <td>Rápido</td>
    <td>Cada semana</td>
  </tr>
  <tr>
    <td>aesgcm</td>
    <td>AES-GCM</td>
    <td>Muy Rápido</td>
    <td>Cada semana</td>
  </tr>
</table>
<br>

# ¿Cómo lo hago?

Es muy sencillo activar el encriptado para *etcd*, en OpenShift lo haremos indicándo al *cluster operator* del *apiserver* que queremos activar el encriptado, para ello editaremos el recurso.

```bash
oc edit apiserver cluster
```

Y dentro del `spec.encryption.type` indicaremos el encriptado que queramos usar, teniendo las siguientes 3 opciones.

```bash
spec:
  encryption:
    type: identity
```

```bash
spec:
  encryption:
    type: aescbc
```

```bash
spec:
  encryption:
    type: aesgcm
```

# Probemos

Vamos a empezar comprobando cómo sin el encriptado podemos recuperar los datos almacenados en *etcd* y luego revisamos el encriptado y que los nuevos recursos creados serán almacenados ya directamente encriptados.

Para estas pruebas, crearemos secretos de *kubernetes* y los consultaremos directamente desde los *pods* donde tenemos en ejecución *etcd*.

Por comodidad y ya que no se trata de pruebas en un entorno productivo, todos los secretos serán creados en el *namespace* *default* del clúster. (Rompiendo con todas las buenas prácticas de uso).

Creamos el primer secreto.

```bash
oc create secret generic secret1 --from-literal secret1key=secret1value -n default
```

Conectamos a uno de los pods de *etcd* y recuperamos el secreto.
```bash
oc get pod -n openshift-etcd

NAME                                                           READY   STATUS      RESTARTS   AGE
etcd-ip-10-0-154-175.eu-west-1.compute.internal                4/4     Running     0          7d1h
etcd-ip-10-0-168-36.eu-west-1.compute.internal                 4/4     Running     0          7d1h
etcd-ip-10-0-198-120.eu-west-1.compute.internal                4/4     Running     0          7d1h
```
```bash
oc exec -it etcd-ip-10-0-154-175.eu-west-1.compute.internal -c etcdctl -n openshift-etcd -- bash

[root@ip-10-0-154-175 /]# etcdctl endpoint status --cluster -w table
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|  https://10.0.168.36:2379 | 52870ba68e6a4d51 |   3.5.6 |  129 MB |      true |      false |        11 |    3941829 |            3941829 |        |
| https://10.0.198.120:2379 | 672326c961e3deca |   3.5.6 |  129 MB |     false |      false |        11 |    3941829 |            3941829 |        |
| https://10.0.154.175:2379 | fb8fce6cd83c2441 |   3.5.6 |  127 MB |     false |      false |        11 |    3941829 |            3941829 |        |
+---------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

Recuperamos el valor que está almacenado en *etcd* para el secreto que hemos creado. Al no tener activado el encriptado, podemos comprobar que se puede obtener toda la información del secreto.

```bash
[root@ip-10-0-154-175 /]# etcdctl get /kubernetes.io/secrets/default/secret1
/kubernetes.io/secrets/default/secret1
k8s


v1Secret�
�
secret1default"*$a0695a95-6ede-41e6-b2f7-84d50d4f27932��ؤ�g
kubectl-createUpdatev��ؤFieldsV1:3
1{"f:data":{".":{},"f:secret1key":{}},"f:type":{}}B

secret1key
           secret1valueOpaque"
```

# Encriptamos a AES-GCM

Habilitamos el encriptado de *etcd*, todos los valores que ya existan en *etcd* serán encriptados también y no únicamente aquellos que sean almacenados después de haber habilitado la encriptación.

Editamos la configuración para el *apiserver* y habilitamos *AES-GCM*.

```bash
oc edit apiserver cluster
```

Quedando de una forma similar a:

```bash
apiVersion: config.openshift.io/v1
kind: APIServer
  name: cluster
spec:
  audit:
    profile: Default
  encryption:
    type: aesgcm
```

Durante el proceso de encriptado podemos comprobar el estado, tanto para el *api* de *kubernetes* como para el *api* de *OpenShift*.

```bash
oc get kubeapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'

EncryptionInProgress
Resource secrets is not encrypted
```

```bash
oc get openshiftapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'

EncryptionInProgress
Resource routes.route.openshift.io is not encrypted
```

Una vez que el proceso termine, ante las mismas comprobaciones, recibiremos las siguientes respuestas, indicando que el proceso ha finalizado y que el encriptado está habilitado.

```bash
EncryptionCompleted
All resources encrypted: secrets, configmaps

----------

EncryptionCompleted
All resources encrypted: routes.route.openshift.io
```

Si ahora comprobamos el secreto que ya teníamos en *etcd* veremos que ya se encuentra encriptado, aprovechamos para crear un segundo secreto, el cual ya se almacena directamente encriptado.

```bash
oc create secret generic secret2 --from-literal secret2key=secret2value -n default
```

```bash
[root@ip-10-0-154-175 /]# etcdctl get /kubernetes.io/secrets/default/secret1
/kubernetes.io/secrets/default/secret1
k8s:enc:aesgcm:v1:1:�DޣM��<a�H6�`�g�P:�K�U�nI�){
      �_賬�_�
          1�P^FR�f&��y�ம?�
oJ2a�SJ�\4�.��_�U������\[����G��Al��B)��=n�[��Hw���xK�f�g��Ƽ#9��4���\1�L���m[�y�q2��Z�c	��Gv�Z��G��a��o6��IP�݋����];��X8	�l7��+�����Nf�/�4b�`嘰��
��TlZ�





[root@ip-10-0-154-175 /]# etcdctl get /kubernetes.io/secrets/default/secret2
/kubernetes.io/secrets/default/secret2
k8s:enc:aesgcm:v1:1aYhZV�e|4@�ڴ����?8W (V�Y,gm@n&�#Ї�L�$��� eM�R�M�c��
~�wBx�ax(�?3�NI7��ĳ �v���]�(���U����D^����R�S�z�	 ��)���s�E�2��_ſ��dW��,U`#��ڼ�S!b$p;��]#��5��8�S�c,�c�n�ͧ����Q��D��w���������"���ʩ����;�3)����y�q���7U�������A%E+
```

# Desactivando el encriptado

Es perfectamente posible desactivar el encriptado, al igual que cuando lo activamos, se trata de un proceso que lleva tiempo, ya que también desencriptará todos los datos ya existentes en *etcd*.

Para desactivarlo, es tan sencillo como indicar que el perfil de encriptado será `identity`.

```bash
apiVersion: config.openshift.io/v1
kind: APIServer
  name: cluster
spec:
  audit:
    profile: Default
  encryption:
    type: identity
```

Realizando las comprobaciones de estado, podremos observar que el desencriptado está en proceso hasta que una vez concluido se indicará que todo está desencriptado.

```bash
oc get openshiftapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'

DecryptionInProgress
Encryption mode set to identity and decryption is not finished

----------
oc get kubeapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'

DecryptionInProgress
Encryption mode set to identity and decryption is not finished
```

```bash
oc get kubeapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="Encrypted")]}{.reason}{"\n"}{.message}{"\n"}'

DecryptionCompleted
Encryption mode set to identity and everything is decrypted
```

Con todo desencriptado, en nuestras consultas a *etcd* volveremos a recuperar los datos en plano.

```bash
[root@ip-10-0-154-175 /]# etcdctl get /kubernetes.io/secrets/default/secret1
/kubernetes.io/secrets/default/secret1
k8s


v1Secret�
�
secret1default"*$a0695a95-6ede-41e6-b2f7-84d50d4f27932��ؤ�g
kubectl-createUpdatev��ؤFieldsV1:3
1{"f:data":{".":{},"f:secret1key":{}},"f:type":{}}B

secret1key
           secret1valueOpaque"






[root@ip-10-0-154-175 /]# etcdctl get /kubernetes.io/secrets/default/secret2
/kubernetes.io/secrets/default/secret2
k8s


v1Secret�
�
secret2default"*$004c405f-5847-4fef-ac57-2d8ac33c92122��ۤ�g
kubectl-createUpdatev��ۤFieldsV1:3
1{"f:data":{".":{},"f:secret2key":{}},"f:type":{}}B

secret2key
           secret2valueOpaque"
```

# Pasando de AES-CBC a AES-GCM

Si ya teníamos encriptado *etcd* en versiones previas de OpenShift, es decir, con AES-CBC habilitado, es tan sencillo como cambiar el perfil a emplear para encriptar *etcd*, pasando de:

```bash
spec:
  encryption:
    type: aescbc
```

a
```bash
spec:
  encryption:
    type: aesgcm
```

Durante el proceso de rotado en las configuraciones, nos encontraremos el mensaje en el *cluster operator* del *apiserver* el mismo mensaje que durante la rotación de las claves de encriptación, el cual se ve como:

```bash
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
...
kube-apiserver                             4.13.1    True        True          False	  7d20h   EncryptionMigrationControllerProgressing: migrating resources to a new write key: [core/secrets]...
...
```

Consiguiendo que los valores que ya estaban encriptados sean reencriptados bajo el nuevo perfil.

# Referencias

<a href="https://docs.openshift.com/container-platform/4.13/security/encrypting-etcd.html#enabling-etcd-encryption_encrypting-etcd" target="_blank">Enabling etcd encryption - Official OCP4.13 Documentation</a><br>
<a href="https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#encrypting-your-data" target="_blank">Encrypting data - Kubernetes Documentation</a>
