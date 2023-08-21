---
layout: post
title:  "Usando IRSA (AWS IAM Roles for Service Accounts) en OpenShift desplegado en modo Mint"
author: danifernandezs
categories: [ ocp4, aws, irsa, cco]
image: assets/images/posts/2023-08-21/irsa_sts_flow.png
featured: false
hidden: false
toc: false
---

Para permitir que las aplicaciones desplegadas accedan a los servicios de AWS, se puede asociar un rol de IAM con una `service account` de Kubernetes usando IRSA (AWS IAM Roles for Service Accounts). <br><br>De esta forma, la `service account` puede otorgar permisos de acceso a servicos de AWS a los contenedores en cualquier pod vinculado con la `service account`.

Hay que recordar las prácticas de seguridad recomendadas por las propia AWS.

- Principio de permisos mínimos: los permisos de IAM se limitan a una service account y solo los pods que usan esa service account tienen acceso a estos permisos de IAM. Esto permite el uso de políticas de IAM detalladas que dan el acceso mínimo a los recursos de AWS.
- Aislamiento de credenciales: un contenedor solo puede obtener credenciales para el rol de IAM que esté asociado con la service account a la que está vinculado. Un contenedor nunca tiene acceso a las credenciales destinadas a otro contenedor.
- Auditoría: el registro de eventos generados al acceder a los recursos de AWS se pueden auditar de forma individual a la service account y su rol asociado.

# Determinando el modo del Cloud Credential Operator

Podemos comprobar el modo en el cual está configurado el Cloud Credential Operator, para ello ejecutamos el siguiente comando:

```bash
oc get cloudcredentials cluster -o=jsonpath={.spec.credentialsMode}
```

En la salida, podemos recibir las siguiente opciones.

- '': El Cloud Credentials Operator está en modo por defecto.
- Mint: El Cloud Credentials Operator está en modo Mint.
- Passthrough: El Cloud Credentials Operator está en modo Passthrough.
- Manual: El Cloud Credentials Operator está en modo Manual.

En el caso del clúster usado para este post y las pruebas, desplegado en AWS, esta comprobación devuelve la siguiente salida.

```bash
$ oc get cloudcredentials cluster -o=jsonpath={.spec.credentialsMode}

$ oc get cloudcredentials cluster -o=jsonpath={.spec}
{"credentialsMode":"","logLevel":"Normal","operatorLogLevel":"Normal"}
```

Lo cual indica que el Cloud Credentials Operator está en modo por defecto, por lo que deberemos comprobar el secreto en el namespace de kube-system para deferminar el estado del Cloud Credentials Operator.

```bash
$ oc get secret aws-creds -n kube-system -o=jsonpath={.metadata.annotations}
{"cloudcredential.openshift.io/mode":"mint"}
```

En este caso, el clúster de OpenShift tiene configurado el Cloud Credentials Operator en modo Mint.

# Flujo de autenticación de las Service Accounts

Basándonos en la imagen oficial del flujo de uso de STS (AWS Security Token Service) en clústeres de OpenShift desplegados en AWS, se requiere en el lado de AWS un IAM Identity Provider desplegado y configurado junto con la configuración para OIDC (OpenID Connect) en un bucket de S3, ya que los tokens temporales serán firmados por este OIDC.

El par de clave pública-privada comparará la parte pública en el lado AWS y la parte privada en el token signer que se encuentra en el clúster de OpenShift.

![Details about the STS flow]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2023-08-21/irsa_sts_flow.png)

En el clúster de OpenShift, para poder hacer uso de las anotaciones en los despliegues y poder indicar el acceso IAM, se requiere el `EKS pod identity mutating webhook`.

Para comprobar que está disponible en nuestro clúster:
```bash
$ oc get mutatingwebhookconfigurations pod-identity-webhook

NAME                   WEBHOOKS   AGE
pod-identity-webhook   1          3d5h
```

# Obtención del binario ccoctl

Para el paso inicial de la creación del `AWS IAM Identity provider` y el bucket S3 con los ficheros de configuración para el OIDC, es necesario el uso del binario de `ccoctl` (aka, cloud credential operator binary).

Este binario lo podemos obtener desde el centro de descargas, al igual que el resto de los binarios, como el binario de `oc`, el instalador, las ISO de CoreOS, etc...

Para el clúster de prueba, que se trata de OpenShift 4.12.25, la ruta es:<br>
<a href="https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.12.25/" target="_blank">https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.12.25/</a><br>

```bash
$ wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.12.25/ccoctl-linux-4.12.25.tar.gz
...
HTTP request sent, awaiting response... 200 OK
Length: 29321008 (28M) [application/x-tar]
Saving to: ‘ccoctl-linux-4.12.25.tar.gz’

ccoctl-linux-4.12.25.tar.gz            100%[=========>]  27.96M  32.6MB/s    in 0.9s    

 - ‘ccoctl-linux-4.12.25.tar.gz’ saved [29321008/29321008]

$ tar xf ccoctl-linux-4.12.25.tar.gz 
```

# Creando los recursos de AWS con el Cloud Credential Operator binary

Para la creación del identity provider debemos indicar la clave pública que se quiere emplear en el IDP y que la parte privada será indicada y desplegada en el clúster de OpenShift.

En nuestro caso, como el clúster ya está desplegado y configurado, ya disponemos de la clave privada configurada en el ServiceAccount Token signer, por lo tanto, lo primero que realizaremos será obtener la parte pública para luego poder crear el IDP.

Extraemos la clave pública del ServiceAccount Token signer de nuestro clúster de OpenShift
```bash
$ oc get configmap --namespace openshift-kube-apiserver bound-sa-token-signing-certs --output json | jq --raw-output '.data["service-account-001.pub"]' > serviceaccount-signer.public

$ cat serviceaccount-signer.public 
-----BEGIN RSA PUBLIC KEY-----
MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEA4ctSxS5wY9iAlTYSmYhl
pGpr4d7WPb500RN4Gq7wCA100vLTgV5PVFannO6On9d3W6V6xNODe/xkqLnBHnXa
i+UJ5VDZwy4iFbNZWJtLQ1jCjGRw+k46XCwOtDRLuf50cr9M+cNF3riIARgAznpJ
hxJv45JXyIAQvJa6CB1dz15kwvqFwdfGqDTzbyYI7ZkD6OUGMVDu3qLe1zWF8SpJ
3yfdsal9kvpvtrDDkdRGKPiQVUrWe2YjZP8YxfqESblwETfpSnjwgMF9f1QqdH1E
pxaBPOIHzWzVR5TAiMIvD0yCxoHZJgsoQaziV+hziZ26XL4nQOe6JxCt7yoFN4nc
Gdzy3TTUZn2LCPgXYZHjSXdjzbzYSirwvwrFfGfPozwz4BzxNmEhgInPYzpG2bQp
zReDYT+CYBqqZWTiUgabUOCscu6U5vSNhl6n6mQdGpQ0F3IDHFx9j9V5ixgTZZMc
ViEeNRDYg2sT8CcGah63rqcrpfqxfQtx1QfVcVMJGlykC4I94XHZ+MI1XjwOJR4V
fJgR8wxzPNO+eQAsPn6UTy4wIDysnmRhUKunQtYqVkEyNVNoUV7jUhxXBgNqur1c
AvQGxhVS1+C4vkgnXRfcF6mVz8fvIze3WiRQYS/qxv1UYy7TzudzgJV/y3VGi//l
ZB1NTZgJLZDWVvaB9CPjk30CAwEAAQ==
-----END RSA PUBLIC KEY-----
```

Como ya disponemos de clave pública al recuperarla del clúster, no es necesario crear un par de claves vía `ccoctl`, por lo que almacenaremos la clave pública en el directorio que comenzaremos a usar para crear el Identity Provider.

```bash
$ mkdir ./enabling_irsa
$ mv serviceaccount-signer.public ./enabling_irsa/serviceaccount-signer.public
```

Creamos el Identity Provider y el bucket S3 con los ficheros de configuración para el OIDC. <br> Debemos seleccionar la región de AWS en la cual desplegar el IDP, decidiendo también el nombre que tendrá el proveedor de identidades.

```bash
$ ./ccoctl aws create-identity-provider --output-dir enabling_irsa --name irsa-mint --region eu-west-1

2023/-- Bucket irsa-mint-oidc created
2023/-- OpenID Connect discovery document in the S3 bucket irsa-mint-oidc at .well-known/openid-configuration updated
2023/-- Reading public key
2023/-- JSON web key set (JWKS) in the S3 bucket irsa-mint-oidc at keys.json updated
2023/-- Identity Provider created with ARN: arn:aws:iam::10xxxxxxxx83:oidc-provider/irsa-mint-oidc.s3.eu-west-1.amazonaws.com
```

De la salida del comando anterior, nos debemos guardar el ARN del Identity Provider que acabamos de crear en AWS. <br> Para nuestro caso nos quedamos con el ARN siguiente.

```bash
arn:aws:iam::10xxxxxxxx83:oidc-provider/irsa-mint-oidc.s3.eu-west-1.amazonaws.com
```

Podemos comprobar en nuestra cuenta de AWS los recursos creados, el bucket S3 y el Identity provider.

![S3 Bucket for OIDC]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2023-08-21/s3bucket.png)<br>
![AWS Identity Provider]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2023-08-21/identityprovider.png)

Ahora deberemos reconfigurar el CR de autenticación del clúster de OpenShift con el ARN del nuevo IDP que acabamos de crear. <br> El propio binario `ccoctl` ha creado el yaml que podemos aplicar con la modificación del ARN a reconfigurar.

Podemos verificar el yaml generado y posteriormente aplicarlo en el clúster.

```bash
$ cat enabling_irsa/manifests/cluster-authentication-02-config.yaml 
apiVersion: config.openshift.io/v1
kind: Authentication
metadata:
  name: cluster
spec:
  serviceAccountIssuer: https://irsa-mint-oidc.s3.eu-west-1.amazonaws.com
```
```bash
$ oc apply -f enabling_irsa/manifests/cluster-authentication-02-config.yaml 
authentication.config.openshift.io/cluster configured
```

Debemos esperar a que todos los pods de `kube-apiserver` sean actualizados con la nueva configuración relacionada con el Identity Provider.

```bash
$ oc get pods -n openshift-kube-apiserver | grep kube-apiserver
kube-apiserver-guard-ip-10-0-157-6.eu-west-1.compute.internal     1/1     Running     3          23d
kube-apiserver-guard-ip-10-0-173-37.eu-west-1.compute.internal    1/1     Running     2          23d
kube-apiserver-guard-ip-10-0-207-182.eu-west-1.compute.internal   1/1     Running     2          23d
kube-apiserver-ip-10-0-157-6.eu-west-1.compute.internal           5/5     Running     0          7d10h
kube-apiserver-ip-10-0-173-37.eu-west-1.compute.internal          5/5     Running     0          7d10h
kube-apiserver-ip-10-0-207-182.eu-west-1.compute.internal         5/5     Running     0          7d10h
```
```bash
$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.12.25   True        True          False      20d     APIServerDeploymentProgressing: deployment/apiserver.openshift-oauth-apiserver: 2/3 pods have been updated to the latest generation
baremetal                                  4.12.25   True        False         False      23d     
cloud-controller-manager                   4.12.25   True        False         False      23d     
cloud-credential                           4.12.25   True        False         False      23d     
cluster-autoscaler                         4.12.25   True        False         False      23d     
config-operator                            4.12.25   True        False         False      23d     
console                                    4.12.25   True        False         False      23d     
control-plane-machine-set                  4.12.25   True        False         False      20d     
csi-snapshot-controller                    4.12.25   True        False         False      23d     
dns                                        4.12.25   True        False         False      23d     
etcd                                       4.12.25   True        False         False      23d     
image-registry                             4.12.25   True        False         False      23d     
ingress                                    4.12.25   True        False         False      12d     
insights                                   4.12.25   True        False         False      23d     
kube-apiserver                             4.12.25   True        True          False      23d     NodeInstallerProgressing: 3 nodes are at revision 12; 0 nodes have achieved new revision 13
kube-controller-manager                    4.12.25   True        False         False      23d     
kube-scheduler                             4.12.25   True        False         False      23d     
kube-storage-version-migrator              4.12.25   True        False         False      23d     
machine-api                                4.12.25   True        False         False      23d     
machine-approver                           4.12.25   True        False         False      23d     
machine-config                             4.12.25   True        False         False      23d     
marketplace                                4.12.25   True        False         False      23d     
monitoring                                 4.12.25   True        False         False      23d     
network                                    4.12.25   True        False         False      23d     
node-tuning                                4.12.25   True        False         False      23d     
openshift-apiserver                        4.12.25   True        False         False      20d     
openshift-controller-manager               4.12.25   True        False         False      23d     
openshift-samples                          4.12.25   True        False         False      12d     
operator-lifecycle-manager                 4.12.25   True        False         False      23d     
operator-lifecycle-manager-catalog         4.12.25   True        False         False      23d     
operator-lifecycle-manager-packageserver   4.12.25   True        False         False      23d     
service-ca                                 4.12.25   True        False         False      23d     
storage                                    4.12.25   True        False         False      23d     
```
```bash
$ oc get pods -n openshift-kube-apiserver | grep kube-apiserver
kube-apiserver-guard-ip-10-0-157-6.eu-west-1.compute.internal     1/1     Running     3          23d
kube-apiserver-guard-ip-10-0-173-37.eu-west-1.compute.internal    1/1     Running     2          23d
kube-apiserver-guard-ip-10-0-207-182.eu-west-1.compute.internal   1/1     Running     2          23d
kube-apiserver-ip-10-0-157-6.eu-west-1.compute.internal           5/5     Running     0          7d10h
kube-apiserver-ip-10-0-173-37.eu-west-1.compute.internal          5/5     Running     0          35s
kube-apiserver-ip-10-0-207-182.eu-west-1.compute.internal         5/5     Running     0          7d10h
```
```bash
$ oc get pods -n openshift-kube-apiserver | grep kube-apiserver
kube-apiserver-guard-ip-10-0-157-6.eu-west-1.compute.internal     1/1     Running     3          23d
kube-apiserver-guard-ip-10-0-173-37.eu-west-1.compute.internal    1/1     Running     2          23d
kube-apiserver-guard-ip-10-0-207-182.eu-west-1.compute.internal   1/1     Running     2          23d
kube-apiserver-ip-10-0-157-6.eu-west-1.compute.internal           5/5     Running     0          5m41s
kube-apiserver-ip-10-0-173-37.eu-west-1.compute.internal          5/5     Running     0          9m35s
kube-apiserver-ip-10-0-207-182.eu-west-1.compute.internal         5/5     Running     0          35s
```
```bash
$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.12.25   True        False         False      20d     
baremetal                                  4.12.25   True        False         False      23d     
cloud-controller-manager                   4.12.25   True        False         False      23d     
cloud-credential                           4.12.25   True        False         False      23d     
cluster-autoscaler                         4.12.25   True        False         False      23d     
config-operator                            4.12.25   True        False         False      23d     
console                                    4.12.25   True        False         False      23d     
control-plane-machine-set                  4.12.25   True        False         False      20d     
csi-snapshot-controller                    4.12.25   True        False         False      23d     
dns                                        4.12.25   True        False         False      23d     
etcd                                       4.12.25   True        False         False      23d     
image-registry                             4.12.25   True        False         False      23d     
ingress                                    4.12.25   True        False         False      12d     
insights                                   4.12.25   True        False         False      23d     
kube-apiserver                             4.12.25   True        False         False      23d     
kube-controller-manager                    4.12.25   True        False         False      23d     
kube-scheduler                             4.12.25   True        False         False      23d     
kube-storage-version-migrator              4.12.25   True        False         False      23d     
machine-api                                4.12.25   True        False         False      23d     
machine-approver                           4.12.25   True        False         False      23d     
machine-config                             4.12.25   True        False         False      23d     
marketplace                                4.12.25   True        False         False      23d     
monitoring                                 4.12.25   True        False         False      23d     
network                                    4.12.25   True        False         False      23d     
node-tuning                                4.12.25   True        False         False      23d     
openshift-apiserver                        4.12.25   True        False         False      20d     
openshift-controller-manager               4.12.25   True        False         False      23d     
openshift-samples                          4.12.25   True        False         False      12d     
operator-lifecycle-manager                 4.12.25   True        False         False      23d     
operator-lifecycle-manager-catalog         4.12.25   True        False         False      23d     
operator-lifecycle-manager-packageserver   4.12.25   True        False         False      23d     
service-ca                                 4.12.25   True        False         False      23d     
storage                                    4.12.25   True        False         False      23d     
```

# Reinicio de todos los pods del clúster

Es necesario un reinicio completo de los pods del clúster ya que todas las ServiceAccount deben ser recargadas debido al ajuste del serviceAccountIssuer del clúster.

Este proceso llevará varios minutos. En el clúster de prueba para este post, formado por 3 nodos en el control plane y 3 nodos de cómputo, sin cargas de trabajo reales, el proceso de reinicio de todos los pods llevó 14 minutos.

```bash
$ for I in $(oc get ns -o jsonpath='{range .items[*]} {.metadata.name}{"\n"} {end}'); \
      do oc delete pods --all -n $I; \
      sleep 1; \
      done
```
```bash
No resources found
pod "activator-7575c84cd5-4q5r2" deleted
pod "activator-7575c84cd5-zw89h" deleted
...
pod "node-resolver-4v9rz" deleted
pod "node-resolver-6l2wn" deleted
pod "node-resolver-6wq26" deleted
pod "node-resolver-nxh4w" deleted
pod "node-resolver-qnvmt" deleted
pod "dns-operator-57c5d979cd-tgczn" deleted
pod "etcd-guard-ip-10-0-157-6.eu-west-1.compute.internal" deleted
...
pod "service-ca-operator-7cfbf6d54d-x6phg" deleted
No resources found
```

# Probando la asignación de roles y la funcionalidad

Para las pruebas, emplearemos el contenedor oficial para la CLI de AWS.

Emplearemos el siguiente despliegue, inicialmente sin referencias a ninguna Service Account, para comprobar que inicialmente no tenemos ningún permiso heredado de los nodos de ejecución.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aws-irsa
  namespace: irsa-test
spec:
  selector:
    matchLabels:
      app: aws
  replicas: 1
  template:
    metadata:
      labels:
        app: aws
    spec:
      containers:
        - name: aws
          image: amazon/aws-cli:2.13.11
          command: 
            - sh
            - -c
            - "while true;do sleep 60;done;"
          env:
          - name: HOME
            value: /tmp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  paused: false
```

Aplicamos el despliegue y comprobamos que el pod no dispone de configuración alguna para acceder a recursos de AWS.

```bash
$ oc apply -f deployment.yaml 
deployment.apps/aws-irsa created

$ oc get po
NAME                        READY   STATUS    RESTARTS   AGE
aws-irsa-577454c596-mwg6p   1/1     Running   0          3s

$ oc exec -t aws-irsa-577454c596-mwg6p -- aws s3 ls

Unable to locate credentials. You can configure credentials by running "aws configure".
command terminated with exit code 253

$ oc exec -t aws-irsa-577454c596-mwg6p -- aws ec2 describe-instances

You must specify a region. You can also configure your region by running "aws configure".
command terminated with exit code 253

$ oc exec -t aws-irsa-577454c596-mwg6p -- aws sts get-caller-identity

Unable to locate credentials. You can configure credentials by running "aws configure".
command terminated with exit code 253
```

## Rol de IAM y asignación

En este punto, ya todo depende de referenciar el rol de IAM que queramos emplear en el despliegue en nuestro clúster. <br> Suponemos la situación más típica, es decir, que el equipo de seguridad (o cualquier otro) será quien crea, mantiene y nos facilita el rol de IAM que deberemos utilizar.

Lo más importante en el rol de IAM es la política de confianza, en la cual haremos referencia al OIDC que hemos creado en la cuenta de AWS.

Ejemplo de la trust relationship, con la referencia al ARN del OIDC creado y el endpoint.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::10xxxxxxxx83:oidc-provider/irsa-mint-oidc.s3.eu-west-1.amazonaws.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "irsa-mint-oidc.s3.eu-west-1.amazonaws.com:sub": "system:serviceaccount:irsa-test:irsa-test"
                }
            }
        }
    ]
}
```

Para las pruebas, creamos un rol de IAM que permita leer los buckets de S3. (Con la política propia de AWS para Read Only)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:Get*",
                "s3:List*",
                "s3:Describe*",
                "s3-object-lambda:Get*",
                "s3-object-lambda:List*"
            ],
            "Resource": "*"
        }
    ]
}
```

Una vez que tenemos el rol creado, tomamos el ARN que deberemos referenciar en la service account con la que desplegaremos el pod.

Nuestro ARN es:

```
arn:aws:iam::10xxxxxxxx83:role/irsa-s3
```

Creamos la service account que utilizará el rol de IAM

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: irsa-test
  namespace: irsa-test
  annotations:
    eks.amazonaws.com/role-arn: "arn:aws:iam::10xxxxxxxx83:role/irsa-s3"
    # optional: Defaults to "sts.amazonaws.com" if not set
    eks.amazonaws.com/audience: "sts.amazonaws.com"
    # optional: When set to "true", adds AWS_STS_REGIONAL_ENDPOINTS env var
    #   to containers
    eks.amazonaws.com/sts-regional-endpoints: "true"
    # optional: Defaults to 86400 for expirationSeconds if not set
    #   Note: This value can be overwritten if specified in the pod
    #         annotation as shown in the next step.
    eks.amazonaws.com/token-expiration: "86400"
```

Momento en el cual reajustamos el despliegue para referenciar a la Service Account.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aws-irsa
  namespace: irsa-test
spec:
  selector:
    matchLabels:
      app: aws
  replicas: 1
  template:
    metadata:
      labels:
        app: aws
    spec:
      containers:
        - name: aws
          image: amazon/aws-cli:2.13.11
          command: 
            - sh
            - -c
            - "while true;do sleep 60;done;"
          env:
          - name: HOME
            value: /tmp
      serviceAccount: irsa-test
      serviceAccountName: irsa-test
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  paused: false
```

Una vez que hemos desplegado el pod, el `EKS pod identity mutating webhook` inyectará las variables de entorno necesarias junto con el punto de montaje necesario para realizar la tarea del `AssumeRoleWithWebIdentity`.

Esto significa, que sobre el despliegue del ejemplo anterior, en el pod mantenido por el deployment se agrega el siguiente contenido:
```yaml
...
      env:
        - name: HOME
          value: /tmp
        - name: AWS_ROLE_ARN
          value: 'arn:aws:iam::10xxxxxxxx83:role/irsa-s3'
        - name: AWS_WEB_IDENTITY_TOKEN_FILE
          value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
...
      volumeMounts:
        - name: aws-iam-token
          readOnly: true
          mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
...
volumes:
    - name: aws-iam-token
      projected:
        sources:
          - serviceAccountToken:
              audience: sts.amazonaws.com
              expirationSeconds: 86400
              path: token
        defaultMode: 420
...
```

Comprobamos nuevamente los permisos, en este caso, nuestro pod ya es capaz de leer los buckets S3 y recibe un identity por parte de AWS para sus peticiones.

```bash
$ oc get po
NAME                        READY   STATUS    RESTARTS   AGE
aws-irsa-547cb6d8c7-9jvsm   1/1     Running   0          14m

$ oc exec -t aws-irsa-547cb6d8c7-9jvsm -- aws sts get-caller-identity
{
    "UserId": "ARXXXXXXXXXXXXXXXXX5I:botocore-session-1692603816",
    "Account": "10xxxxxxxx83",
    "Arn": "arn:aws:sts::10xxxxxxxx83:assumed-role/irsa-s3/botocore-session-1692603816"
}

$ oc exec -t aws-irsa-547cb6d8c7-9jvsm -- aws ec2 describe-instances

You must specify a region. You can also configure your region by running "aws configure".
command terminated with exit code 253

$ oc exec -t aws-irsa-547cb6d8c7-9jvsm -- aws s3 ls
2023-07-28 12:57:19 irsa-7n9rz-image-registry-eu-west-1-wmsdynvplrwpeodtgypyutqwgj
2023-08-20 18:00:51 irsa-mint-oidc
```

# Referencias

<a href="https://docs.openshift.com/container-platform/4.12/authentication/managing_cloud_provider_credentials/about-cloud-credential-operator.html#cco-determine-mode-cli_about-cloud-credential-operator" target="_blank">Determining the Cloud Credential Operator mode by using the CLI - Official OCP4.12 Documentation</a><br>
<a href="https://github.com/openshift/cloud-credential-operator/blob/master/docs/sts.md" target="_blank">Short lived Credentials with AWS Security Token Service - GitHub</a><br>
<a href="https://github.com/aws/amazon-eks-pod-identity-webhook" target="_blank">EKS pod identity mutating webhook</a><br>
<a href="https://docs.openshift.com/container-platform/4.12/authentication/managing_cloud_provider_credentials/cco-mode-sts.html#sts-mode-create-aws-resources-ccoctl_cco-mode-sts" target="_blank">Creating AWS resources with the Cloud Credential Operator utility</a><br>
<a href="https://hub.docker.com/r/amazon/aws-cli" target="_blank">Amazon Web Services Command Line Interface (AWS CLI)
</a><br>
