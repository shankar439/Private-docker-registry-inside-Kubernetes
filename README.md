<h1 align="center"> Kubernetes </h1>
<h2 align="center"> Hosting a Private Docker registry in kubernetes</h2>

![imagegit](https://github.com/shankar439/Images/assets/70714976/2b7ca6dd-a6cc-4498-b89f-84977efaa5a5)
### Benifits of Hosting our own Docker Registry in Kubernetes.
  - choose wisely accroding to your requirement
  - ### Pros
    - Private and Secure from anonymous person
    - We get a Private Docker Registry free of cost rather purchasing.
    - Rather sticking with plans we can create as much registry and allocate to different Team, we have unlimited pull, push.
  - ### Cons
    - We do not get the UI to interact with our registry. we need to manually look into directory to check everything.

### prerequisite
  - Kubernetes Cluster 
  - Ingress Controller deployed in cluster
  - Cert Manager 
  - Domain Name 

## Docker

- Before Deploying Docker registry in kubernetes, Firts we need to create UserName and Password.
- This username and password are Credentials for the Docker Registry, remember the credentials which we will use later to login to the Registry.

```
docker run --entrypoint htpasswd httpd:2 -Bbn youruser yourpassword
```
- `NOTE: Make sure to replace the username and password as you want`
- Run the above command to get a output something like this  <mark>shankar:$2y$05$NTnMqkHw68ZtQD3ZPtilTOzv/S/slkqWTaRvjDCyaq/Cp.v9kyB2K</mark>


### Password file

- Create a file `'docker.password'` and paste the output which we created using the docker command. Later this file will be attached to the Docker Registry.

<br>
<br>


## Volume
- ## [Local Storage Class](https://kubernetes.io/docs/concepts/storage/storage-classes/#local)

  - As the Storage need to be Persistent I choose Local Storage Class type.

### locla-storage-class.yaml
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
```
kubectl -n docker-registry apply -f local-storage-class.yaml
```

- ## [Persistent Volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
  - NOTE: Before applying persistent volume make sure you created the Directory `'/var/docker-registry/'`
### local-pv-registry.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-registry
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /var/docker-registry/
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - yourServerHostname
``` 
```
kubectl -n docker-registry apply -f local-pv-registry.yaml
```

- ## [Persistent Volume Claim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
  - NOTE: Before applying Persistent Volume Claim first apply Persistent Volume.
### registry-pvc.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata: 
  name: registry-pvc
spec: 
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: local-storage
  volumeName: local-pv-registry
```
```
kubectl -n docker-registry apply -f registry-pvc.yaml
```

<br>
<br>


## Deploy Docker Registry in Kubernetes Cluster

### First lets create a namespace for the Docker registry.

```
kubectl create namespace docker-registry
```
<br>

### Create a secret for the Docker registry
- Change the the directory where you have the file `'docker.password'` and apply the below command.
```
kubectl create secret generic registry-auth-secret --from-file=docker.password -n docker-registry
```

### Registry-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: docker-registry
  labels:
    app: registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
      - name: registry
        resources:
          requests:
            memory: "100Mi"
            cpu: "250m"
          limits:
            memory: "200Mi"
            cpu: "500m"
        image: registry:2
        ports:
        - containerPort: 5000
        volumeMounts:
        - name: repo-vol
          mountPath: "/var/lib/registry"
        - name: certs-vol
          mountPath: "/certs"
          readOnly: true
        - name: auth-vol
          mountPath: "/auth"
          readOnly: true
        env:
        - name: REGISTRY_AUTH
          value: "htpasswd"
        - name: REGISTRY_AUTH_HTPASSWD_REALM
          value: "Registry Realm"
        - name: REGISTRY_AUTH_HTPASSWD_PATH
          value: "/auth/registry.password"
        - name: REGISTRY_HTTP_TLS_CERTIFICATE
          value: "/certs/tls.crt"
        - name: REGISTRY_HTTP_TLS_KEY
          value: "/certs/tls.key"
        - name: VIRTUAL_HOST
          value: "registry.shankar.com"
     
      volumes:
      - name: repo-vol
        persistentVolumeClaim:
          claimName: registry-pvc
      - name: certs-vol
        secret:
          secretName: registry-tls
      - name: auth-vol
        secret:
          secretName: registry-auth-secret
---
apiVersion: v1
kind: Service
metadata:
  name: docker-registry
  namespace: docker-registry
spec:
  selector:
    app: registry
  ports:
  - port: 5000
    targetPort: 5000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: docker-registry
  namespace: docker-registry
  labels:
    app: registry
  annotations:
    cert-manager.io/cluster-issuer: "lets-encrypt"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-body-size: 1024m
spec:
  tls:
  - hosts:
    - registry.shankar.com
    secretName: registry-tls
  rules:
  - host: registry.shankar.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: docker-registry
            port:
              number: 5000
```
```
kubectl -n docker-registry apply -f registry-deployment.yaml
```

- NOTE: As this is example name make sure to replace the Domain name to yours `'registry.shankar.com'`.
- Mase sure everything Deployed Correctly without Error using
```
kubectl -n docker-registry get po,svc,ingress
```

## Test Docker Registry

- If everything is working fine lets login to our Own Private Docker Registry.

```
docker login registry.shankar.com
```
- After this you will prompt to enter username and password one after another. This the part we will use our credential to login.
```
Username: shankar
Password: dodopasword
```

# END
