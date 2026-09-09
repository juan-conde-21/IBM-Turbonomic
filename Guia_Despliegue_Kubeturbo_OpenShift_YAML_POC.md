# Guía de despliegue Kubeturbo en Red Hat OpenShift mediante YAML

**Producto:** IBM Turbonomic Kubeturbo\
**Versión:** 8.20.6\
**Método:** YAML\
**Perfil inicial:** Reader (`turbo-cluster-reader`)

------------------------------------------------------------------------

# 1. Objetivo

Este documento describe el procedimiento para desplegar Kubeturbo en un
clúster Red Hat OpenShift utilizando el manifiesto YAML oficial de IBM
Turbonomic.

La implementación inicial considera un despliegue controlado para una
fase de POC:

-   Despliegue mediante YAML (sin Operator).
-   Permisos de solo lectura.
-   Validación de descubrimiento del clúster.
-   Control inicial de consumo de recursos.

Posteriormente, dependiendo del alcance requerido (por ejemplo ejecución
de acciones Move/Resize), se podrá evaluar el cambio hacia un perfil con
mayores permisos.

------------------------------------------------------------------------

# 2. Alcance inicial

El YAML utilizado corresponde al perfil:

    turbo-cluster-reader

Este perfil permite:

-   Descubrimiento de recursos OpenShift/Kubernetes.
-   Recolección de información del clúster.
-   Generación de recomendaciones en Turbonomic.

No habilita inicialmente:

-   Movimiento de pods.
-   Modificación automática de workloads.
-   Ejecución de acciones.

------------------------------------------------------------------------

# 3. Prerrequisitos

## 3.1 Información requerida

  Parámetro            Descripción
  -------------------- ----------------------------------------
  Turbonomic Server    URL HTTPS del servidor
  Versión Turbonomic   Debe coincidir con la imagen Kubeturbo
  Client ID            OAuth 2.0 generado en Turbonomic
  Client Secret        OAuth 2.0 generado en Turbonomic
  Nombre del Target    Nombre del clúster OpenShift

------------------------------------------------------------------------

# 4. Parámetros a modificar en el YAML

Antes de aplicar el manifiesto se deben actualizar los siguientes
valores.

------------------------------------------------------------------------

## 4.1 Namespace

Buscar:

``` yaml
namespace: turbonomic
```

Validar que el namespace corresponda al definido para la instalación.

------------------------------------------------------------------------

## 4.2 Credenciales Turbonomic

Ubicación:

``` yaml
kind: Secret
```

Modificar:

``` yaml
clientid: <Client_id_encoded_base64>

clientsecret: <Client_secret_encoded_base64>
```

Los valores deben estar codificados en Base64:

``` bash
echo -n "valor" | base64
```

------------------------------------------------------------------------

## 4.3 Servidor Turbonomic

Ubicación:

``` yaml
turbo.config
```

Modificar:

``` json
"turboServer": "<https://Turbo_Server_URL_or_IP_address>"
```

Ejemplo:

``` json
"turboServer": "https://turbonomic.company.local"
```

------------------------------------------------------------------------

## 4.4 Nombre del clúster

Modificar:

``` json
"targetName":"<Your_Cluster_Name>"
```

Este valor será utilizado como nombre del Target dentro de Turbonomic.

------------------------------------------------------------------------

## 4.5 Imagen Kubeturbo

Validar:

``` yaml
image: icr.io/cpopen/turbonomic/kubeturbo:8.20.6
```

La versión debe coincidir con la versión del servidor Turbonomic.

------------------------------------------------------------------------

# 5. Control de recursos del Deployment

Para la primera fase de validación se recomienda agregar límites
explícitos al contenedor.

Ubicación:

``` yaml
kind: Deployment

containers:
- name: kubeturbo
```

Agregar:

``` yaml
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    memory: "2Gi"
```

Consideraciones:

-   Se utiliza `500m` de CPU inicialmente para una POC controlada.
-   Se evita definir `cpu limit` para prevenir throttling.
-   La memoria podrá ajustarse según cantidad de pods y workloads
    administrados.

------------------------------------------------------------------------

# 6. Despliegue

Crear namespace:

``` bash
oc create namespace turbonomic
```

Aplicar manifiesto:

``` bash
oc apply -f kubeturbo_reader_full.yaml
```

------------------------------------------------------------------------

# 7. Validaciones

## Validar pod

``` bash
oc get pods -n turbonomic
```

Resultado esperado:

    kubeturbo-xxxxx   1/1   Running

------------------------------------------------------------------------

## Revisar logs

``` bash
oc logs deployment/kubeturbo -n turbonomic
```

Validar:

-   Conexión con Turbonomic.
-   Discovery correcto.
-   Sin errores RBAC.

------------------------------------------------------------------------

## Revisar consumo

``` bash
oc adm top pod -n turbonomic
```

Registrar consumo inicial durante la POC.

------------------------------------------------------------------------

# 8. YAML completo de despliegue

El siguiente bloque corresponde al YAML utilizado para el despliegue
inicial.

Antes de aplicar, realizar las modificaciones indicadas en la sección
anterior.

``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  # Update the namespace value if required
  name: turbo-user
  namespace: turbonomic
---
#option to use secret for Turbo credentials
apiVersion: v1
kind: Secret
metadata:
  name: turbonomic-credentials
  namespace: turbonomic
type: Opaque
data:
  # username: <Username_encoded_base64>
  # password: <Password_encoded_base64>
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
  # use this yaml to create a binding that will assign cluster-admin to your turbo ServiceAccount 
  # Provide a value for the binding name: and update namespace if needed
  # The name should be unique for Kubeturbo instance
  name: turbo-all-binding-kubeturbo-turbo
  namespace: turbonomic
subjects:
- kind: ServiceAccount
  # Provide the correct value for service account name: and namespace if needed
  name: turbo-user
  namespace: turbonomic
roleRef:
  # User creating this resource must have permissions to add this policy to the SA
  kind: ClusterRole
# for other limited cluster admin roles, see samples provided
  name: turbo-cluster-reader
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: ConfigMap
metadata:
  # use this yaml to provide details kubeturbo will use to connect to the Turbo Server
  # requires Turbo Server and kubeturbo pod 6.4.3 and higher 
  # Provide a value for the config name: and update namespace if needed
  name: turbo-config
  namespace: turbonomic
data:
  # Update the values for version, turboServer, opsManagerUserName, opsManagerPassword
  # For version, use Turbo Server Version, even when running CWOM
  # The opsManagerUserName requires Turbo administrator role
  #
  # For targetConfig, targetName provides better group naming to identify k8s clusters in UI
  # - If no targetConfig is specified, a default targetName will be created from the apiserver URL in
  #   the kubeconfig.
  # - Specify a targetName only will register a probe with type Kubernetes-<targetName>, as well as
  #   adding your cluster as a target with the name Kubernetes-<targetName>.
  # - Specify a targetType only will register a probe without adding your cluster as a target.
  #   The probe will appear as a Cloud Native probe in the UI with a type Kubernetes-<targetType>.
  #
  # Define node groups by node role, and automatically enable placement policies to limit to 1 per host
  # DaemonSets are identified by default. Use daemonPodDetectors to identify by name patterns using regex or by namespace.
  #
  # serverMeta.proxy format for authenticated and non-authenticated "http://username:password@proxyserver:proxyport or http://proxyserver:proxyport"
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
                "version": "8.20.6",
                "turboServer": "<https://Turbo_Server_URL_or_IP_address>"
            },
            "restAPIConfig": {
                "turbonomicCredentialsSecretName": "turbonomic-credentials"
            }
        },
        "targetConfig": {
            "targetName":"<Your_Cluster_Name>"
        },
        "HANodeConfig": {
            "nodeRoles": [ "master"]
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  # use this yaml to deploy the kubeturbo pod 
  # Provide a value for the deploy/pod name: and update namespace if needed
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
      # If using a private registry, specify the image pull secret name here
      # imagePullSecrets:
      #  - name: <your-image-pull-secret-name>
      #
      # Update serviceAccount if needed
      serviceAccount: turbo-user
      #
      # Assigning Kubeturbo to node, see 
      # https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/ 
      #
      # nodeSelector:
      #   kubernetes.io/hostname: worker0
      #
      # Or, use affinity:
      #
      # affinity:
      #   nodeAffinity:
      #       requiredDuringSchedulingIgnoredDuringExecution:
      #         nodeSelectorTerms:
      #         - matchExpressions:
      #           - key: kubernetes.io/hostname
      #             operator: In
      #             values:
      #             - worker1
      #
      # Or, use taints and tolerations
      #
      # tolerations:
      # - key: "key1"
      #   operator: "Equal"
      #   value: "mytaint"
      #   effect: "NoSchedule"
      securityContext:
        runAsNonRoot: true
      containers:
      - name: kubeturbo
        # Replace the image version with matching Turbo Server version such as 8.13.0
        image: icr.io/cpopen/turbonomic/kubeturbo:8.20.6
        env:
        - name: KUBETURBO_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        # Set SKIP_TAG_CHECK to "true" to skip tag check during probe upgrade
        - name: SKIP_TAG_CHECK
          value: "false"
        args:
        - --turboconfig=/etc/kubeturbo/turbo.config
        - --v=2
        # Comment out the following two args if running in k8s 1.10 or older, or
        # change to https=false and port=10255 if unsecure kubelet read only is configured
        - --kubelet-https=true
        - --kubelet-port=10250
        # SECURITY: Set to true to use the node proxy endpoint for kubelet connections
        - --use-node-proxy-endpoint=false
        # Uncomment for pod moves in OpenShift
        #- --scc-support=*
        # Uncomment for pod moves with pvs
        #- --fail-volume-pod-moves=false
        # Uncomment to override default, and specify your own location
        #- --busybox-image=docker.io/busybox
        # or uncomment below to pull from RHCC
        #- --busybox-image=registry.access.redhat.com/ubi9/ubi-minimal
        # Uncomment to specify the secret name which holds the credentials to busybox image
        #- --busybox-image-pull-secret=<secret-name>
        # Specify nodes to exclude from cpu frequency getter job.
        # Note kubernetes.io/os=windows and/or beta.kubernetes.io/os=windows labels will be automatically excluded by default.
        # If specified all the labels will be used to select the node ignoring the default.
        #- --cpufreq-job-exclude-node-labels=kubernetes.io/key=value
        # The complete cpufreqgetter image uri used for fallback node cpu frequency getter job.
        #- --cpufreqgetter-image=icr.io/cpopen/turbonomic/cpufreqgetter
        # The cpufreqgetter image tag, valid for Kubeturbo version 8.16.5+ and the valid options are the Turbo released version after 8.16.5 or using latest otherwise.
        #- --cpufreqgetter-image-tag=latest
        # The name of the secret that stores the image pull credentials for cpufreqgetter image.
        #- --cpufreqgetter-image-pull-secret=<secret-name>
        # Uncomment to stitch using IP, or if using Openstack, Hyper-V/VMM
        #- --stitch-uuid=false
        # Uncomment to customize readiness retry threshold. Kubeturbo will try readiness-retry-threshold times before giving up. Default is 60. The retry interval is 10s.
        #- --readiness-retry-threshold=60
        # Uncomment to disable the cleanup of the resources which are created by kubeturbo for the scc impersonation.
        #- --cleanup-scc-impersonation-resources=false
        # Uncomment to skip creating the resources the scc impersonation
        #- --skip-creating-scc-impersonation-resources=true
        # [ArgoCD integration] The email to be used to push changes to git
        #- --git-email=""
        # [ArgoCD integration] The username to be used to push changes to git
        #- --git-username=""
        # [ArgoCD integration] The name of the secret which holds the git credentials
        #- --git-secret-name""
        # [ArgoCD integration] The namespace of the secret which holds the git credentials
        #- --git-secret-namespace=""
        # [ArgoCD integration] The commit mode that should be used for git action executions. One of {request|direct}. Defaults to direct
        #- --git-commit-mode=""
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
        # volume will be created, any name will work and must match below
        - name: turbo-volume
          mountPath: /etc/kubeturbo
          readOnly: true
        - name: turbonomic-credentials-volume
          # This mount path cannot be changed
          mountPath: /etc/turbonomic-credentials
          readOnly: true
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: turbo-volume
        configMap:
         # Update configMap name if needed
          name: turbo-config
      - name: turbonomic-credentials-volume
        secret:
          defaultMode: 420
          optional: true
          # Update secret name if needed
          secretName: turbonomic-credentials
      - name: varlog
        emptyDir: {}
      restartPolicy: Always
# this is to create a role with permissions to update the kubeturbo deployment 
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
# If using an image pull secret. Uncomment the section below and change the pull-secret-name to the secret used.
  # - apiGroups: [""]
  #  resources: ["secrets"]
  #  resourceNames: ["<your-image-pull-secret-name>"]          # change name of pull secret here
  #  verbs: ["get", "list"]
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

------------------------------------------------------------------------

# 9. Evolución hacia permisos completos

Para una fase posterior donde se requiera:

-   Move de pods.
-   Resize.
-   Ejecución automática de acciones.

se deberá evaluar el cambio hacia el YAML con permisos administrativos
mínimos requerido por Turbonomic.

La ampliación de permisos se realizará únicamente después de validar:

-   Discovery correcto.
-   Consumo del agente.
-   Requerimiento funcional del cliente.

------------------------------------------------------------------------

# 10. Rollback

Eliminar recursos creados:

``` bash
oc delete -f kubeturbo_reader_full.yaml
```

Eliminar namespace:

``` bash
oc delete namespace turbonomic
```
