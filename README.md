# ST0263 - Tópicos Especiales de Telemática

## Integrantes:
- David González Tamayo (dgonzalez2@eafit.edu.co)
- Juan Esteban Castro García (jecastrog@eafit.edu.co)
- Tomás Bernal Zuluaga (tbernalz@eafit.edu.co)

## Profesor
- **Nombre:** Edwin Nelson Montoya Munera
- **Correo:** emontoya@eafit.edu.co

# Proyecto 2

## 1. Breve descripción de la actividad

En este proyecto 2 basandonos de anteriores [RETO3](https://github.com/dgonzalezt2/reto3-st0263) y [RETO4](https://github.com/dgonzalezt2/reto4-st0263) implementaremos una aplicación de Wordpress utilizando MicroK8s en un entorno de Google Cloud Platform (GCP). Configuramos un clúster con un nodo maestro y nodos de trabajo en MicroK8s, permitiendo así la creación de un clúster de Kubernetes personalizado. Además, establecimos una máquina dedicada como servidor NFS para garantizar la persistencia de los datos de Wordpress y la base de datos en caso de fallos en los pods.

### 1.1. Que aspectos cumplió o desarrolló de la actividad propuesta por el profesor (requerimientos funcionales y no funcionales)

Aspectos cumplidos de la actividad propuesta por el profesor:
· Se realizará la implementación en GCP

· Este proyecto 2, instalará un clúster Kubernetes con el software microk8s (https://microk8s.io/

· El servicio debe implementar un balanceador de cargas, alta disponibilidad en la capa de aplicación, alta disponibilidad en la capa de base de datos y alta disponibilidad en la capa de almacenamiento.

· Debe permitir el aumento dinámico de nodos en el clúster kubernetes, ideal de forma automática, sino manual.

· Para el caso de la base de datos, debe desplegar en kubernetes una base de datos de alta disponibilidad.

· Para el caso del sistema de archivos, desplegar el servidor NFS en el propio clúster para los servicios State full set.

· Implementar un dominio para el servicio. https://proyecto2.dominio.tld

### 1.2. Que aspectos NO cumplió o desarrolló de la actividad propuesta por el profesor (requerimientos funcionales y no funcionales)

* Certificado muestra problemas de dualidad 

## 2. Información general de diseño de alto nivel, arquitectura, patrones, mejores prácticas utilizadas.

Se utilizó la misma arquitectura presentada en los retos:

Arquitectura [Reto3-4](https://github.com/dgonzalezt2/reto3-st0263):

* ![image](https://github.com/dgonzalezt2/reto4-st0263/assets/81880494/a8af5c96-dfba-4f82-bcc6-078d4a51ad31)

## 3. Descripción del ambiente de desarrollo y técnico: lenguaje de programación, librerias, paquetes, etc, con sus numeros de versiones.

Configuración de Certificados SSL
certificate.yaml
```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: certificate-proyecto2
  namespace: default
spec:
  secretName: certificate-proyecto2-mb8l2
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: proyecto2.reto3.me
  dnsNames:
  - proyecto2.reto3.me
```

ssl-letsencrypt-prod.yaml
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: tomasbernalzuluaga@gmail.com
    privateKeySecretRef:
       name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: public
```

ssl-letsencrypt-staging.yaml
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: tomasbernalzuluaga@gmail.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: public
```

Configuración de Ingress
ssl-ingress-routes.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-routes
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - proyecto2.reto3.me
    secretName: certificate-proyecto2-mb8l2
  rules:
  - host: proyecto2.reto3.me
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wordpress
            port:
              number: 80
```
ingress.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
  labels:
    app: wordpress
spec:
  rules:
  - http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: wordpress
            port:
              number: 80
```

Configuración de MySQL
mysql.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-csi
  nfs:
    server: 10.128.0.5
    path: /srv/nfs

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  labels:
    app: mysql
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 5Gi

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - name: mysql
        image: docker.io/bitnami/mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: admin
        - name: MYSQL_DATABASE
          value: microk8s-db
        - name: MYSQL_USER
          value: user1
        - name: MYSQL_PASSWORD
          value: admin
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc

---

apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
```

Configuración de WordPress
wordpress.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: wordpress-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-csi
  nfs:
    server: 10.128.0.5
    path: /srv/nfs

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-pvc
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 5Gi

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql
        - name: WORDPRESS_DB_PASSWORD
          value: admin
        - name: WORDPRESS_DB_USER
          value: user1
        - name: WORDPRESS_DB_NAME
          value: microk8s-db
        - name: WORDPRESS_DEBUG
          value: "1"
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wordpress-pvc

---

apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: wordpress
    tier: frontend
```

Configuración de Almacenamiento
sc-nfs.yaml
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 10.128.0.5
  share: /srv/nfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - hard
```

pvc-nfs.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: nfs-csi
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 5Gi
```


## 4. Descripción del ambiente de EJECUCIÓN (en producción) lenguaje de programación, librerias, paquetes, etc, con sus numeros de versiones.

IP externa: 34.134.49.116

![image](https://github.com/dgonzalezt2/proyecto2-st0263/assets/81880494/1642214a-7592-4c2a-a490-543882c0ec85)

![image](https://github.com/dgonzalezt2/proyecto2-st0263/assets/81880494/cc05a203-5607-4251-85d9-01b5a1759d04)

![image](https://github.com/dgonzalezt2/proyecto2-st0263/assets/81880494/9820e477-15ce-4f2d-8e39-bd8b395e0c4b)

## 5. Resultado final.

[Video sustentacion](https://youtu.be/hUAS_AwMFrA)

Dominio Proyecto 2: https://proyecto2.reto3.me/

![image](https://github.com/dgonzalezt2/proyecto2-st0263/assets/81880494/b55c1ccd-57a0-401b-bda9-21ae7d804500)
![image](https://github.com/dgonzalezt2/proyecto2-st0263/assets/81880494/9908974a-95fa-4542-8655-7b73ff3c7f32)


## 6. Referencias.

* [Microk8s](https://microk8s.io)
* [Usar certificados SSL administrados por Google](https://cloud.google.com/kubernetes-engine/docs/how-to/managed-certs#gcloud)
* [KUBERNETES De NOVATO a PRO! (CURSO COMPLETO EN ESPAÑOL)](https://www.youtube.com/watch?v=DCoBcpOA7W4)
* [Wordpress High Availability on Kubernetes](https://medium.com/@icheko/wordpress-high-availability-on-kubernetes-f6c0bcc2f28d)
* [WordPress on Kubernetes Cluster — Step-by-Step Guide](https://engr-syedusmanahmad.medium.com/wordpress-on-kubernetes-cluster-step-by-step-guide-749cb53e27c7)
* [HIGHLY AVAILABLE WORDPRESS ON KUBERNETES](https://matthewdavis.io/highly-available-wordpress-on-kubernetes/)
* [How to deploy WordPress on Kubernetes — Part 1](https://medium.com/codex/how-to-deploy-wordpress-on-kubernetes-part-1-62cc5bd74410)
* [How to deploy WordPress on Kubernetes — Part 2](https://medium.com/codex/how-to-deploy-wordpress-on-kubernetes-part-2-df1cc9cbaa2e)





