# OpenShift GitOps Lab

## Instalar GitOps Operator

### Identificar el channel a utilizar:

```sh
$ oc get packagemanifests.packages.operators.coreos.com -n openshift-marketplace openshift-gitops-operator -o jsonpath='{.status.defaultChannel}{"\n"}'
```

La salida debería mostrar uno o más valores, como:

```sh
$  oc get packagemanifests.packages.operators.coreos.com -n openshift-marketplace openshift-gitops-operator -o jsonpath='{.status.defaultChannel}{"\n"}'

latest
```

### Crear las Subscriptions:

Utilizando este valor de canal, cree el manifiesto de `Subscription` para instalar el **OpenShift GitOps Operator** como se muestra a continuación. En lugar de crear un nuevo `namespace` y `operatorgroup`, se puede utilizar el `namespace` creado por defecto llamado `openshift-operators`, ya que el **GitOps Operator** soporta "All Namespace" como objetivos:

```sh
$ cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### Autorizar la ServiceAccount de GitOps:

Tras la instalación, el **OpenShift GitOps Operator** crea el `namespace` `openshift-gitops` y una `service account` llamada `openshift-gitops-argocd-application-controller` en ese `namespace`. Esta `serviceaccount` se utiliza para aplicar los manifiestos en el `cluster`. Aunque esta cuenta tiene privilegios suficientes en el `namespace` `openshift-gitops`, no tiene privilegios en otros `namespaces`.

Dado que queremos crear varios recursos con el objetivo de entender las aplicaciones de **GitOps** y cómo funcionan, procederemos a otorgar a esta `service account` privilegios de nivel `cluster-admin`.

> **NOTA:** En un entorno de producción, es posible que deba ser más restrictivo con el `role` que vincula a esta `service account` y los `namespaces` a los que le da acceso.

Use el siguiente comando para otorgar a esta `service account` el rol de `cluster-admin` en todo el clúster:

```sh
$ oc create clusterrolebinding gitops-scc-binding --clusterrole cluster-admin  --serviceaccount openshift-gitops:openshift-gitops-argocd-application-controller
```

### Verificar si el Operator está correctamente instalado:

Se pueden utilizar los siguientes dos comandos para asegurar que la instalación del `operator` ha sido aceptada y ha sido exitosa:

```sh
$ oc get operators openshift-gitops-operator.openshift-operators 
NAME                                            AGE
openshift-gitops-operator.openshift-operators   2m40s
```

```sh
$ oc get csv -n openshift-operators | grep gitops
openshift-gitops-operator.v1.12.0   Red Hat OpenShift GitOps   1.12.0    openshift-gitops-operator.v1.11.2   **Succeeded**
```

Adicionalmente, para verificar si el `deployment` del `operator` es saludable, use lo siguiente:

```sh
$ oc get deployment -n openshift-gitops
```

La salida debería mostrar que todos los `deployments` se están ejecutando correctamente:

```sh
$ oc get deployment -n openshift-gitops
NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
cluster                                      1/1     1            1           8m10s
kam                                          1/1     1            1           8m10s
openshift-gitops-applicationset-controller   1/1     1            1           8m7s
openshift-gitops-dex-server                  1/1     1            1           8m10s
openshift-gitops-redis                       1/1     1            1           8m8s
openshift-gitops-repo-server                 1/1     1            1           8m8s
openshift-gitops-server                      1/1     1            1           8m7s

```

## Acceder a la GUI del GitOps Operator:

Para acceder a la **GUI**, necesitará una **URL** y credenciales. Estas pueden obtenerse de la siguiente manera:

### Obtener Routes:

Para encontrar la **URL** de acceso, use el siguiente comando:

```sh
$ oc get -n openshift-gitops routes openshift-gitops-server -o jsonpath='{.status.ingress[].host}{"\n"}'

```

La salida será similar a:

> openshift-gitops-server-openshift-gitops.apps.cluster-jl57k.dynamic.redhatworkshops.io

### Obtener Secret:

El nombre de usuario por defecto es `admin`, y la contraseña para ese usuario administrador se puede recuperar usando lo siguiente:

```sh
$ oc get -n openshift-gitops secrets openshift-gitops-cluster -o jsonpath='{.data.admin\.password}' | base64 -d ; echo

```

La salida será similar a:

> DVqetLgGo30UCNb1pm4X8JnAPQShRyEx

### ArgoCD GUI:

Acceda a la **ArgoCD GUI** utilizando la información anterior. La interfaz debería verse así:

## GitOps - La primera vez:

El **GitOps Operator** define un **CR** (Custom Resource) llamado `Applications`. Las `Applications` se utilizan para sincronizar el `cluster` en ejecución con los manifiestos definidos en **Git**. La siguiente figura describe los campos en la definición del **CR** `Application`:

> **TIP:** **OpenShift GitOps Operator** es la versión comercial de **ArgoCD**. Por lo tanto, los términos se utilizan indistintamente.

### Su primera aplicación:

Vamos a crear una aplicación sencilla que creará un `namespace` y ejecutará un `pod` en él. Los manifiestos para estos ya han sido definidos [aquí](https://github.com/jovemfelix/ocp-gitops/tree/main/manifests/set0). Todo lo que la aplicación debe hacer es apuntar al repositorio y dejar que **GitOps** haga su magia. Cree la aplicación como se muestra aquí:

```sh
$ cat << EOF | oc apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
  name: first-app
  namespace: openshift-gitops
spec:
  destination:
    server: 'https://kubernetes.default.svc'
  project: default
  source:
    directory:
      recurse: true
    path: manifests/set0
    repoURL: https://github.com/jovemfelix/ocp-gitops.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    retry:
      limit: 20
EOF

```

### Verificar el estado de la aplicación usando el comando OC:

Una vez creada la aplicación, puede ver el progreso de su estado como se muestra aquí:

```sh
$ oc get apps -n openshift-gitops first-app 
NAME        SYNC STATUS   HEALTH STATUS
first-app   Synced        Progressing

## pausar unos segundos
$ oc get apps -n openshift-gitops first-app 
NAME        SYNC STATUS   HEALTH STATUS
first-app   Synced        Healthy

```

El estado `Healthy` de la aplicación indica que el `namespace` y el `pod` se han creado correctamente. Confirme esto de la siguiente manera:

```sh
$ oc get pods -n first-gitops-space
NAME   READY   STATUS    RESTARTS   AGE
pod    1/1     Running   0          4m15s

```

Puede intentar eliminar este `pod`, pero como **Git** es la fuente de verdad (*Source of Truth*) y la definición del `pod` aún existe en **Git**, el `pod` se recreará:

```sh
$ oc delete pods -n first-gitops-space pod 
pod "pod" deleted

$ oc get pods -n first-gitops-space
NAME   READY   STATUS    RESTARTS   AGE
pod    1/1     Running   0          3s

```

### Verificar el estado de la aplicación usando la GUI:

El estado de la aplicación y los **CRs** que creó pueden verificarse fácilmente en la **GUI**:

## Orden de aplicación de CRs - ArgoCD SyncWaves:

El ejemplo anterior fue simple. Sin embargo, incluso en este conjunto básico de **CRs**, debe haber una secuencia para aplicarlos con éxito. Si **GitOps** intenta crear el `Pod` antes de crear el `namespace`, fallará.
El **GitOps Operator** sabe que ciertas cosas van en un orden específico y se encarga de ello. Sin embargo, en muchos casos (como en recursos personalizados o **CRDs**) no podrá tomar esa decisión. En esos casos, puede guiarlo asociando un número al manifiesto. Este número, llamado **SyncWave**, ayuda al `operator` a determinar la secuencia para aplicar los diversos manifiestos que ve en **Git**.

Además, el `operator` puede categorizar los manifiestos en tres `Phases`: **PreSync**, **Sync** y **PostSync**. Por defecto, todo está en la categoría **Sync**. Los manifiestos en cada categoría se ordenan individualmente según su **SyncWave** (el valor por defecto es 0).

Combinando estos mecanismos, el **GitOps operator** utiliza el siguiente orden para determinar cómo aplicar los manifiestos:

* La **Phase** en la que se encuentran (**Pre-Sync**, **Sync** o **Post-Sync**). Los manifiestos de una fase deben aplicarse con éxito antes de pasar a la siguiente.
* El valor de **SyncWave** definido en la anotación del recurso, del valor más bajo al más alto.
* Si ambos coinciden, el desempate es el valor del **CR** o `kind`. Primero `Namespaces`, luego `Services`, luego `Deployments`, etc. (esto es lo que permitió que nuestra primera aplicación funcionara sin guía manual).
* El desempate final, si todo lo demás coincide, es el nombre del manifiesto en orden ascendente.

### Más sobre Sync Phases:

La fase se define en un manifiesto bajo `.metadata.annotations.argocd.argoproj.io/hook`. Ejemplo:

```yaml
metadata: 
  annotations: 
    argocd.argoproj.io/hook: PreSync

```

El propósito de los tipos de fase es:

* **PreSync**: Tiene prioridad para aplicarse antes que los recursos en la fase **Sync**.
* **Sync**: Los recursos se aplican después de que todas las aplicaciones en **PreSync** se han aplicado con éxito y alcanzado un estado `Healthy`.
* **PostSync**: Se ejecutan después de una sincronización exitosa (ej. notificaciones por email).

La siguiente figura demuestra este concepto visualmente:

(Figura obtenida de este )

También existe una fase **SyncFail** para recursos que se aplican si la sincronización falla.

### Más sobre SyncWaves:

Todos los manifiestos tienen un `wave` de cero por defecto. El valor se puede establecer usando `metadata.annotations.argocd.argoproj.io/sync-wave`. El valor también puede ser negativo:

```yaml
metadata:
    annotations:
      argocd.argoproj.io/sync-wave: "-203"

```

El **GitOps Operator** (**ArgoCD**) aplicará el valor más bajo y se asegurará de que devuelva un estado `healthy` antes de continuar. **Argo CD** no aplicará el siguiente manifiesto hasta que el anterior reporte salud.

La siguiente figura demuestra este concepto visualmente:

(Figura obtenida de este )

## Crear una aplicación para demostrar SyncWaves:

Para crear una aplicación que demuestre los **Sync Waves**, use lo siguiente:

```sh
$ mkdir ~/gitops
$ cd ~/gitops
$ cat << EOF > app1.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
  labels:
    app.kubernetes.io/instance: ebc-multicloud-gitops-hub
  name: experiment-app1
  namespace: openshift-gitops
spec:
  destination:
    server: 'https://kubernetes.default.svc'
  project: default
  source:
    directory:
      recurse: true
    path: manifests/set1
    repoURL: https://github.com/jovemfelix/ocp-gitops.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    retry:
      limit: 20
EOF

```

## Ejecutar la aplicación demostrando SyncWaves:

Esta aplicación demostrará el uso de **SyncWaves**. Los manifiestos llamados están definidos con los siguientes valores:

| manifest | kind | name | namespace | Phase | SyncWave | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| ns1.yaml | Namespace | argotest1-1 |  | Sync | 200 |  |
| sa1.yaml | ServiceAccount | cli-job-sa | argotest1-1 | Sync | 201 |  |
| role1.yaml | ClusterRoleBinding | cli-job-sa-argotest1-1-rolebinding |  | Sync | 202 |  |
| job1.yaml | Job | testjob-1-1 | argotest1-1 | Sync | 203 | Este job tardará 100 segundos en finalizar |
| powerpod2.yaml | Namespace | argotest1-2 |  | Sync | 300 | Este recurso deberá esperar a que "testjob-1-1" complete |
| powerpod2.yaml | ServiceAccount | cli-job-sa | argotest1-2 | Sync | 300 |  |
| powerpod2.yaml | ClusterRoleBinding | cli-job-sa-argotest1-2-rolebinding | argotest1-2 | Sync | 302 |  |
| powerpod2.yaml | Job | testjob1-2 | argotest1-2 | Sync | 303 |  |

Ejecute la aplicación con el siguiente comando:

```sh
$ oc apply -f app1.yaml 

```

Observe que la aplicación aparece en la **GUI** de **ArgoCD**:

La imagen muestra que la aplicación aún no se ha sincronizado porque todos los recursos no se han aplicado con éxito. Esto se debe a que `testjob1-1` todavía se está ejecutando y no ha alcanzado un estado `Healthy` (su estado se mostrará como `Progressing`).

Revise los logs de `testjob1-1` y verá que el contador sigue corriendo:

Una vez que el contador llegue a 10, el `job` se completará y se aplicará el siguiente recurso (el `namespace` `argotest1-2`).

Finalmente, la aplicación alcanza un estado de sincronización completa:

Verifique el estado mediante **CLI**:

```sh
$ oc get applications.argoproj.io -n openshift-gitops experiment-app1

```

> NAME                       SYNC STATUS   HEALTH STATUS
> 
> 
> 
> 
> experiment-app1      Synced              Healthy

## Ejecutar aplicaciones para demostrar Phase Hooks:

Para demostrar el uso de `Phases`, crearemos otra aplicación:

```sh
$ cat << EOF > app2.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  annotations:
  labels:
    app.kubernetes.io/instance: ebc-multicloud-gitops-hub
  name: experiment-app2
  namespace: openshift-gitops
spec:
  destination:
    server: 'https://kubernetes.default.svc'
  project: default
  source:
    directory:
      recurse: true
    path: manifests/set2
    repoURL: https://github.com/jovemfelix/ocp-gitops.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    retry:
      limit: 20
EOF

```

### Ejecutar la segunda aplicación:

Esta tabla refleja el orden de ejecución basado en **Phase Hooks** y **Sync Waves**:

| manifest | kind | name | namespace | Phase | SyncWave | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| ns.yaml | Namespace | argotest2 |  | PreSync | 1 |  |
| presync1.yaml | Job | presync1 | argotest2 | PreSync | 103 | El job tardará 100 segundos |
| presync2.yaml | Job | presync2 | argotest2 | PreSync | 203 | Se ejecutará ANTES que testjob1 por el hook PreSync |
| powerpod.yaml | Job | testjob1 | argotest2 | Sync | 103 | presync2 nunca terminará, por lo que este job nunca iniciará |

Ejecute el `job` usando:

```sh
$ oc apply -f app2.yaml 

```

En la **GUI**, verá que `presync1` se está ejecutando. Aunque `testjob1` tiene un valor de **SyncWave** menor, `presync2` tendrá preferencia por estar en la fase **PreSync**.

Tras completar el conteo, el siguiente `job` iniciará:

Como `presync2` tiene un bucle infinito, el `job` `testjob1` nunca comenzará. Como resultado, la aplicación nunca alcanzará un estado `Healthy`.

```sh
$ oc get applications.argoproj.io -n openshift-gitops experiment-app2 

```

> NAME                        SYNC STATUS   HEALTH STATUS
> 
> 
> 
> 
> experiment-app2      OutOfSync              Missing

---
