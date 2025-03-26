# Laboratorio Introductorio: RBAC en Kubernetes

## Objetivo

Comprender cómo funcionan los controles de acceso RBAC en Kubernetes mediante un laboratorio práctico que simula una situación real de permisos restringidos.

---

## Requisitos

- Un clúster de Kubernetes funcional (Minikube, Kind, Rancher Desktop, etc.)
- Acceso a kubectl
- Conocimientos básicos de namespaces y pods

---

## Escenario

Queremos que un usuario limitado (ServiceAccount llamada alumno) pueda **ver los pods** de un namespace (test-rbac), pero **no pueda hacer nada más**.

---

## Recursos creados

| Recurso              | Descripción                                |
|----------------------|--------------------------------------------|
| Namespace: test-rbac | Espacio de prueba para este ejercicio     |
| ServiceAccount: alumno | Representa a nuestro usuario limitado |
| Role: rol-ver-pods     | Permite ver pods (get, list)      |
| RoleBinding: alumno-ver-pods | Asocia el rol a la cuenta alumno |

---

## Paso a paso

### 1. Crear el namespace y la ServiceAccount


kubectl create namespace test-rbac
kubectl create serviceaccount alumno -n test-rbac


### 2. Crear un pod que use la ServiceAccount


kubectl apply -f pod-alumno.yaml


Contenido de pod-alumno.yaml:

apiVersion: v1
kind: Pod
metadata:
  name: pod-alumno
  namespace: test-rbac
spec:
  serviceAccountName: alumno
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]

### 3. Probar que no tiene permisos aún


kubectl auth can-i list pods --as=system:serviceaccount:test-rbac:alumno -n test-rbac

Resultado esperado: no


### 4. Crear el Role que permite ver pods


kubectl apply -f role-ver-pods.yaml


Contenido de role-ver-pods.yaml:

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: test-rbac
  name: rol-ver-pods
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

---

### 5. Asociar el Role a la ServiceAccount con un RoleBinding


kubectl apply -f rolebinding-alumno.yaml


Contenido de rolebinding-alumno.yaml:

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alumno-ver-pods
  namespace: test-rbac
subjects:
- kind: ServiceAccount
  name: alumno
  namespace: test-rbac
roleRef:
  kind: Role
  name: rol-ver-pods
  apiGroup: rbac.authorization.k8s.io

---

### 6. Verificar que ahora sí puede ver pods


kubectl auth can-i list pods --as=system:serviceaccount:test-rbac:alumno -n test-rbac


Resultado esperado: yes

---

### 7. Verificar que **NO puede hacer más de lo permitido**


kubectl auth can-i delete pods --as=system:serviceaccount:test-rbac:alumno -n test-rbac
kubectl auth can-i get secrets --as=system:serviceaccount:test-rbac:alumno -n test-rbac

Resultado esperado: no en ambos casos

---

Lecciones clave

- Una ServiceAccount no puede hacer nada sin permisos explícitos.
- Roles controlan los permisos dentro de un namespace.
- RoleBindings asignan esos permisos a usuarios o cuentas de servicio.
- Puedes simular cualquier identidad con --as.

---