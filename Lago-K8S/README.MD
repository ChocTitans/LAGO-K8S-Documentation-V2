### **Lago deployment in K8s**

We will start by deploying PostgreSQL DB and Redis. First of all, we need to set up a PVC/PV/SC for both of them. To achieve this, follow the steps below:

Create an EBS volume in EC2 Edit the volume ID in `lago-postgres-data-pvc-pv-sc.yaml` and `lago-redis-data-pv-pvc.yaml` as shown below:

## Requirements

- EBS CSI DRIVER `Will be added in CICD` [@OfficialDocumentation](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)

### **For Postgres :**

<img src="./assets/imgs/VolumeID.png" width="100%">

File : `lago-postgres-data-pvc-pv-sc.yaml` (Line 23)

```
  storageClassName: lago-postgres-data-storageclass
  awsElasticBlockStore:
    volumeID: 	vol-XXXXXXXXXXX
    fsType: ext4
```

```
kubectl apply -f .\lago-postgres-data-pvc-pv-sc.yaml
```

Apply the PostgreSQL deployment and service manifests:

```
kubectl apply -f .\db-deployment.yaml
kubectl apply -f .\db-service.yaml
```
---

### **For Redis (same steps as PostgreSQL):**

```
  persistentVolumeReclaimPolicy: Retain
  storageClassName: lago-redis-data-storageclass
  awsElasticBlockStore:
    volumeID: 	vol-XXXXXXXXXXX
    fsType: ext4

```

Apply the Redis deployment and service manifests:

```
kubectl apply -f .\redis-deployment.yaml
kubectl apply -f .\redis-service.yaml
```

---

### **Front Deployment/service :**

```
kubectl apply -f .\front-deployment.yaml
kubectl apply -f .\front-service.yaml  
```

To port forward the frontend service:

```
kubectl port-forward -n default service/front 80:80
```

----------------------------------------------------------------------------------------------------------------------

### **Api Deployment/service :**

Create 2 EBS Volume with your desired Storage Size then edit the volume ID in both `lago-apiworker-data-pvc-pv.yaml` and `lago-storage-api-data-pvc-pv.yaml` as needed.

```
kubectl apply -f .\lago-storage-api-data-pvc-pv.yaml
kubectl apply -f .\lago-apiworker-data-pvc-pv.yaml 
```

Apply the API deployment and service manifests:

```
kubectl apply -f .\api-deployment.yaml   
kubectl apply -f .\api-service.yaml
```

### **For Api-Worker/Clock :**

Apply the API-Worker and Clock deployment manifests:


```
kubectl apply -f .\api-clock-deployment.yaml
kubectl apply -f .\api-worker-deployment.yaml
```

For the RSA_PRIVATE_KEY **All of them**

First you have to generate an RSA with :

```
openssl genpkey -algorithm RSA -out private_key.pem
```

Open the file and remove the `-----BEGIN PRIVATE KEY-----` and `-----END PRIVATE KEY-----`

```
openssl base64 -in private_key.pem -out private_key_base64.txt
```

Create a `secret.yaml`

```
apiVersion: v1
kind: Secret
metadata:
  name: my-rsa-secret
type: Opaque
data:
  API_SECRET : <put the value of private_key_base64>
```

In every of these file `api-clock-deployment.yaml` `api-deployment.yaml` `api-worker-deployment.yaml`, instead of this :

```
- name: LAGO_RSA_PRIVATE_KEY
  value: <rsa_KEY>
```

Load the secret that you have just added :

```
  - name: LAGO_RSA_PRIVATE_KEY
    valueFrom:
      secretKeyRef:
        name: my-rsa-secret
        key: API_SECRET
```


### **For PDF/Network policy:**

```
kubectl apply -f .\lago-default-networkpolicy.yaml
kubectl apply -f .\pdf-deployment.yaml
```


To test if everything is working :

```
kubectl port-forward -n default service/front 80:80
kubectl port-forward -n default service/api 3000:3000
```
----------------------------------------------------------------------------------------------------------------------------

For the ingress, you must install ALB : [@ALBIngress](https://github.com/TrouveTaVoie/eks-cluster)

```
kubectl apply -f .\ingress-front.yaml
```

once you add the certificate / domain name, make sure to edit the LAGO_API_URL and LAGO_FRONT_URL in : `Api-deployment`, `Api-Clock-deployment`, `api-worker-deployment`.


---

### **For the snapshot Life-cycle**

To not lose any data in case of deletion of cluster etc..

Create a lifecycle for the volume created for each deployment :

<img src="./assets/imgs/Lifecycle.png" width="100%">

Do next in the first page, then for the second page :

<img src="./assets/imgs/Step2.png" width="100%">

You can choose when to start taking snapshots with the following options:

- Daily
- Weekly
- Monthly
- Yearly

Regarding the UTC timing, if you choose 09:00, it will be equivalent to 10:00 GMT+1 (Morocco time).

Now, let's understand the Retention Type: It determines the number of snapshots that will be kept over time. For instance, if you choose daily snapshots starting at 9:00 UTC and select a Retention Type count of 2:

On the first day, a snapshot will be taken at 9:00 UTC.
On the second day, another snapshot will be taken at 9:00 UTC, and the first one will be retained.
On the third day, a new snapshot will be taken at 9:00 UTC, and the oldest snapshot (the first one) will be deleted, keeping only the second and third snapshots.
This retention pattern ensures that the system maintains the specified number of snapshots while replacing the oldest one with each new snapshot taken beyond that count


<img src="./assets/imgs/Snapshot.png" width="100%">

Once a snapshot is created through the lifecycle management, you have the option to use it to create a volume or image. Additionally, you can attach this volume to a Persistent Volume (PV) in your Kubernetes cluster.



Furthermore, the created snapshot can be shared with other AWS accounts. To do this, you need to specify the AWS account ID of the target account.
<img src="./assets/imgs/share.png" width="100%">

Once shared, the target account will be able to access and find the shared snapshot under EC2 > EBS > Snapshots > Private Snapshots section. This allows the target account to utilize the snapshot for their own purposes or create volumes/images from it as needed

<img src="./assets/imgs/private.png" width="20%">
