# Guía de despliegue controlado de KubeTurbo en Red Hat OpenShift mediante YAML

**Producto:** IBM Turbonomic KubeTurbo  
**Método:** YAML directo  
**Perfil inicial:** Reader (`turbo-cluster-reader`)  
**Versión de referencia del ejemplo:** 8.21.1  
**Objetivo:** desplegar KubeTurbo en OpenShift con permisos de lectura y recursos definidos desde el inicio.

> El YAML completo está incluido en esta guía para que pueda utilizarse como referencia directa. Antes de aplicarlo, se deben reemplazar versión, URL, credenciales y nombre del clúster. Para otra versión de Turbonomic se recomienda descargar el `kubeturbo_reader_full.yaml` oficial de esa misma versión y comparar los cambios de RBAC.

---

# 1. Qué hace KubeTurbo

KubeTurbo es el componente que conecta un clúster Kubernetes/OpenShift con Turbonomic.

Permite:

- descubrir nodos, pods, namespaces y workload controllers;
- leer capacidad y utilización;
- enviar información del clúster a Turbonomic;
- generar recomendaciones de optimización;
- ejecutar acciones cuando posteriormente se habilitan los permisos correspondientes.

Para la primera fase se recomienda utilizar **Reader**. De esta manera se valida discovery, consumo y recomendaciones sin ejecutar cambios sobre el clúster.

---

# 2. Capacidad inicial

Antes de desplegar se debe revisar el tamaño del clúster.

Para un ambiente de hasta **5.000 pods** y **5.000 workload controllers**, la configuración inicial utilizada en esta guía es:

| Recurso | Valor |
|---|---:|
| Réplicas | `1` |
| CPU request | `1000m` |
| CPU limit | **No definir** |
| Memory request | `1Gi` |
| Memory limit | `4Gi` |

IBM recomienda no establecer CPU limit para KubeTurbo para evitar throttling.

## 2.1 Tabla de sizing de memoria

| Pods | Workload controllers | Memory limit |
|---:|---:|---:|
| 5.000 | 2.500 | 4 Gi |
| 5.000 | 5.000 | 4 Gi |
| 10.000 | 5.000 | 6 Gi |
| 10.000 | 10.000 | 6.5 Gi |
| 20.000 | 10.000 | 9.2 Gi |
| 20.000 | 20.000 | 12 Gi |
| 30.000 | 15.000 | 13 Gi |
| 30.000 | 30.000 | 16 Gi |

Referencia IBM:  
https://www.ibm.com/docs/en/tarm/8.x?topic=requirements-kubeturbo-resource-limits

---

# 3. Revisar el tamaño del clúster

Cantidad de pods:

```bash
oc get pods -A --no-headers | wc -l
```

Conteo inicial de controllers:

```bash
oc get deployments,statefulsets,daemonsets -A --no-headers | wc -l
```

Registrar:

| Dato | Valor |
|---|---|
| Versión Turbonomic | |
| Versión OpenShift | |
| Pods | |
| Workload controllers | |
| Memory limit seleccionado | |
| Namespace | `turbonomic` |
| Target name | |

Si el clúster supera 5.000 pods/controllers, ajustar el `memory limit` usando la tabla anterior.

---

# 4. Prerrequisitos

Se requiere:

- acceso `oc`;
- permisos para crear ServiceAccount, ClusterRole, ClusterRoleBinding, ConfigMap, Secret, Deployment, Role y RoleBinding;
- Turbonomic disponible por HTTPS;
- OAuth Client ID y Client Secret;
- acceso al OpenShift API Server;
- acceso desde KubeTurbo a kubelets por TCP 10250;
- acceso a `icr.io` o a una registry privada.

---

# 5. Validar versión

La versión de KubeTurbo debe coincidir con la versión de Turbonomic.

Ejemplo utilizado:

```text
Turbonomic : 8.21.1
KubeTurbo  : 8.21.1
```

Para otra versión:

```bash
export TURBO_VERSION="<VERSION_TURBONOMIC>"
```

Y descargar el manifiesto oficial:

```bash
curl -fL   -o kubeturbo_reader_full.yaml   "https://raw.githubusercontent.com/IBM/turbonomic-container-platform/${TURBO_VERSION}/kubeturbo/yamls/kubeturbo_reader_full.yaml"
```

Repositorio IBM:  
https://github.com/IBM/turbonomic-container-platform

---

# 6. Valores que se deben modificar en el YAML

Antes de aplicar el ejemplo, cambiar estos valores.

## 6.1 Credenciales

Generar Base64:

```bash
printf '%s' "<CLIENT_ID>" | base64 -w0
echo
printf '%s' "<CLIENT_SECRET>" | base64 -w0
echo
```

Reemplazar:

```yaml
clientid: <Client_id_encoded_base64>
clientsecret: <Client_secret_encoded_base64>
```

## 6.2 Servidor Turbonomic

Reemplazar:

```json
"turboServer": "https://turbonomic.example.local"
```

## 6.3 Nombre del clúster

Reemplazar:

```json
"targetName":"ocp-cluster-01"
```

## 6.4 Versión

Si no se utiliza 8.21.1, modificar **los dos puntos**:

```json
"version": "8.21.1"
```

y:

```yaml
image: icr.io/cpopen/turbonomic/kubeturbo:8.21.1
```

## 6.5 Recursos

Para hasta 5.000 pods/controllers:

```yaml
resources:
  requests:
    cpu: "1000m"
    memory: "1Gi"
  limits:
    memory: "4Gi"
```

No agregar CPU limit.

---

# 7. YAML completo de referencia

Este es el manifiesto Reader completo utilizado como guía, con el bloque `resources` ya incorporado.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: turbo-user
  namespace: turbonomic
---
apiVersion: v1
kind: Secret
metadata:
  name: turbonomic-credentials
  namespace: turbonomic
type: Opaque
data:
  clientid: <Client_id_encoded_base64>
  clientsecret: <Client_secret_encoded_base64>
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: turbo-cluster-reader
rules:
  - apiGroups:
      - ""
      - apps
      - app.k8s.io
      - apps.openshift.io
      - batch
      - extensions
      - devops.turbonomic.io
      - config.openshift.io
      - argoproj.io
    resources:
      - nodes
      - pods
      - deployments
      - replicasets
      - replicationcontrollers
      - services
      - endpoints
      - namespaces
      - limitranges
      - resourcequotas
      - persistentvolumes
      - persistentvolumeclaims
      - applications
      - jobs
      - cronjobs
      - statefulsets
      - daemonsets
      - deploymentconfigs
      - operatorresourcemappings
      - clusterversions
      - rollouts
    verbs:
      - get
      - watch
      - list
  - apiGroups:
      - machine.openshift.io
    resources:
      - machines
      - machinesets
    verbs:
      - get
      - list
  - apiGroups:
      - ""
    resources:
      - nodes/spec
      - nodes/stats
      - nodes/metrics
      - nodes/proxy
    verbs:
      - get
  - apiGroups:
      - storage.k8s.io
    resources:
      - csinodes
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - policy.turbonomic.io
    resources:
      - slohorizontalscales
      - containerverticalscales
      - policybindings
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - kubevirt.io
    resources:
      - virtualmachineinstances
      - virtualmachines
      - virtualmachineinstancepresets
      - virtualmachineinstancereplicasets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - charts.helm.k8s.io
    resources:
      - kubeturbos
    verbs:
      - get
      - list
      - patch
      - update
      - watch
  - apiGroups:
      - cdi.kubevirt.io
    resources:
      - datasources
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - instancetype.kubevirt.io
    resources:
      - virtualmachineclusterpreferences
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - hco.kubevirt.io
    resources:
      - hyperconvergeds
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - charts.helm.k8s.io
    resources:
      - xls
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: turbo-all-binding-kubeturbo-turbo
  namespace: turbonomic
subjects:
- kind: ServiceAccount
  name: turbo-user
  namespace: turbonomic
roleRef:
  kind: ClusterRole
  name: turbo-cluster-reader
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: turbo-config
  namespace: turbonomic
data:
  turbo-autoreload.config: |-
    {
      "logging": {
        "level": 2
      },
      "nodePoolSize": {
        "min": 1,
        "max": 1000
      },
      "systemWorkloadDetectors": {
        "namespacePatterns": ["kube-.*","openshift-.*","cattle.*"]
      },
      "exclusionDetectors": {
        "operatorControlledWorkloadsPatterns": [],
        "operatorControlledNamespacePatterns": [],
        "oomRecoveryExcludedWorkloads": []
      },
      "daemonPodDetectors": {
        "namespaces": [],
        "podNamePatterns": []
      },
      "policySettings": {
        "containers": {
          "sidecars": {"disableActionGeneration": false}
        }
      },
      "virtualizationConfig": {
        "vmLifecycleWaitTimeoutInMins": 30
      },
      "oomEventHandler": {
        "enableEventWrites": false,
        "enableRecoveryActions": true
      },
      "skipNodeSuspendOnPodEviction": false
    }
  turbo.config: |-
    {
        "communicationConfig": {
            "serverMeta": {
                "version": "8.21.1",
                "turboServer": "https://turbonomic.example.local"
            },
            "restAPIConfig": {
                "turbonomicCredentialsSecretName": "turbonomic-credentials"
            }
        },
        "targetConfig": {
            "targetName":"ocp-cluster-01"
        },
        "HANodeConfig": {
            "nodeRoles": [ "master"]
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeturbo
  namespace: turbonomic
spec:
  replicas: 1
  selector:
     matchLabels:
       app.kubernetes.io/name: kubeturbo
  strategy:
    type: Recreate
  template:
    metadata:
      annotations:
        kubeturbo.io/monitored: "false"
      labels:
        app.kubernetes.io/name: kubeturbo
    spec:
      # Si se utiliza una registry privada:
      # imagePullSecrets:
      # - name: <image-pull-secret>
      serviceAccount: turbo-user
      securityContext:
        runAsNonRoot: true
      containers:
      - name: kubeturbo
        image: icr.io/cpopen/turbonomic/kubeturbo:8.21.1

        # Sizing inicial para hasta 5.000 pods / 5.000 workload controllers.
        # Ajustar memory limit según la tabla de sizing del procedimiento.
        # No definir CPU limit para evitar throttling.
        resources:
          requests:
            cpu: "1000m"
            memory: "1Gi"
          limits:
            memory: "4Gi"

        env:
        - name: KUBETURBO_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: SKIP_TAG_CHECK
          value: "false"
        args:
        - --turboconfig=/etc/kubeturbo/turbo.config
        - --v=2
        - --kubelet-https=true
        - --kubelet-port=10250
        - --use-node-proxy-endpoint=false

        # Opciones que deben habilitarse solo si el caso de uso lo requiere.
        # - --scc-support=*
        # - --fail-volume-pod-moves=false
        # - --busybox-image=registry.access.redhat.com/ubi9/ubi-minimal
        # - --stitch-uuid=false
        # - --readiness-retry-threshold=60

        securityContext:
          privileged: false
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        resizePolicy:
        - resourceName: memory
          restartPolicy: RestartContainer
        volumeMounts:
        - name: turbo-volume
          mountPath: /etc/kubeturbo
          readOnly: true
        - name: turbonomic-credentials-volume
          mountPath: /etc/turbonomic-credentials
          readOnly: true
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: turbo-volume
        configMap:
          name: turbo-config
      - name: turbonomic-credentials-volume
        secret:
          defaultMode: 420
          optional: true
          secretName: turbonomic-credentials
      - name: varlog
        emptyDir: {}
      restartPolicy: Always
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: turbo-user-role
  namespace: turbonomic
rules:
  - apiGroups:
      - apps
    resources:
      - deployments
    resourceNames:
      - kubeturbo
    verbs:
      - get
      - watch
      - update
      - patch
      - list
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: turbo-user-role-binding
  namespace: turbonomic
subjects:
  - kind: ServiceAccount
    name: turbo-user
roleRef:
  kind: Role
  name: turbo-user-role
  apiGroup: rbac.authorization.k8s.io
```

---

# 8. Crear namespace

Si no existe:

```bash
oc create namespace turbonomic
```

---

# 9. Validar antes de aplicar

Guardar el YAML como:

```text
kubeturbo_reader_full_controlado.yaml
```

Validar sintaxis:

```bash
oc apply --dry-run=client -f kubeturbo_reader_full_controlado.yaml
```

Validar contra el API Server:

```bash
oc apply --dry-run=server -f kubeturbo_reader_full_controlado.yaml
```

Revisar que no queden placeholders:

```bash
grep -n "<" kubeturbo_reader_full_controlado.yaml
```

Antes de continuar confirmar:

- namespace correcto;
- URL de Turbonomic correcta;
- credenciales cargadas;
- target name correcto;
- versión correcta en ConfigMap e imagen;
- requests/limits correctos;
- sin CPU limit.

---

# 10. Desplegar

```bash
oc apply -f kubeturbo_reader_full_controlado.yaml
```

Validar:

```bash
oc get pods -n turbonomic
```

Resultado esperado:

```text
NAME                         READY   STATUS    RESTARTS
kubeturbo-xxxxxxxxxx-xxxxx   1/1     Running   0
```

---

# 11. Revisar consumo

```bash
oc adm top pods -n turbonomic
```

Registrar:

| Momento | CPU | Memoria | Reinicios |
|---|---:|---:|---:|
| 15 minutos | | | |
| 1 hora | | | |
| 24 horas | | | |
| 72 horas | | | |

Verificar los recursos aplicados:

```bash
oc get deployment kubeturbo -n turbonomic   -o jsonpath='{.spec.template.spec.containers[0].resources}'
echo
```

---

# 12. Revisar logs

```bash
oc logs deployment/kubeturbo -n turbonomic --tail=200
```

Validar:

- conexión con Turbonomic;
- discovery;
- ausencia de errores RBAC;
- acceso a kubelets;
- ausencia de reinicios/OOM.

---

# 13. Validar en Turbonomic

Revisar:

```text
Settings -> Target Configuration
```

Confirmar:

- target visible;
- nodos descubiertos;
- namespaces descubiertos;
- workloads descubiertos;
- recomendaciones generadas.

En esta fase Reader no se busca ejecutar acciones.

---

# 14. Criterios de aceptación

La instalación inicial queda validada cuando:

- KubeTurbo está `Running`;
- target visible en Turbonomic;
- discovery correcto;
- sin errores persistentes de RBAC;
- sin OOM;
- consumo estable;
- recomendaciones disponibles;
- sin ejecución de acciones.

---

# 15. Evolución a acciones

Después de validar discovery y consumo, se puede evaluar un perfil con permisos adicionales para habilitar, según el caso:

- resize;
- move de pods;
- acciones sobre nodos;
- automatización.

No ampliar permisos durante la validación inicial si todavía no se ha confirmado el comportamiento del agente.

---

# 16. Rollback

Guardar evidencia:

```bash
oc get all -n turbonomic -o wide
oc logs deployment/kubeturbo -n turbonomic > kubeturbo-before-rollback.log
```

Eliminar:

```bash
oc delete -f kubeturbo_reader_full_controlado.yaml
```

Eliminar namespace únicamente si fue creado de forma exclusiva para KubeTurbo:

```bash
oc delete namespace turbonomic
```

---

# 17. Referencias

IBM - Kubeturbo resource limits  
https://www.ibm.com/docs/en/tarm/8.x?topic=requirements-kubeturbo-resource-limits

IBM - Kubeturbo deployment requirements  
https://www.ibm.com/docs/en/tarm/8.x?topic=targets-kubeturbo-deployment-requirements

IBM - Deploying Kubeturbo through YAML  
https://www.ibm.com/docs/en/tarm/8.x?topic=clusters-deploying-kubeturbo-through-yaml

IBM - Turbonomic Container Platform  
https://github.com/IBM/turbonomic-container-platform

---

## Nota

El YAML completo se incluye para facilitar la ejecución y servir como referencia durante una sesión con el cliente.

Para nuevas versiones de Turbonomic, el manifiesto oficial de IBM de esa versión debe seguir siendo la fuente principal. Antes de reutilizar este ejemplo se debe comparar el RBAC y los parámetros con el `kubeturbo_reader_full.yaml` correspondiente al release instalado.
