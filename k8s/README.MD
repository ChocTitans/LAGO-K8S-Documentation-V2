### **Lago deployment in K8s**

### **Requirements**

- EBS CSI DRIVER `Will be added in IaC` [@OfficialDocumentation](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)



### **Kustomization**

We're using kustomization to deploy all the ressources, in the main root there's `kustomization.yaml` that specifies every deployment/service/pvc folder and also using it for secrets/configmap.


**Why too many kustomization file**

We have a file in the root called kustomization.yaml and it specifies the folder that has file manifests.
Let's say we have 5 folder and each of them has X file of manifests, to apply all of them we'll have to `cd` to each folder and use `kustomize build . | kubectl apply -f -` so instead of this, we'll create a folder in the root that calls all the kustomizations in each folder and we won't need to `cd` to all of them. we'll just execute it in the root and all the manifests will be applied

---

Add them to your repository's secrets : 

- Go to your repository's settings.

- Under the settings, navigate to the 'Secrets' section.

- Click on 'New repository secret' or 'Add a secret environment' button.

- For the name, enter `LAGO_API_URL` and the other variables.

- Input the new master key value in the 'Value' field.

These secrets and configmap has to be defined in a workflow `Github Actions` : 

```
      - name: Create ConfigMap file
        run: |
          cd Lago-K8S
          echo "LAGO_API_URL=${{ env.LAGO_API_URL }}" > configmap.env
          echo "LAGO_FRONT_URL=${{ env.LAGO_FRONT_URL }}" >> configmap.env
          echo "LAGO_REDIS_URL=${{ env.LAGO_REDIS_URL }}" >> configmap.env
          echo "LAGO_PDF_URL=${{ env.LAGO_PDF_URL }}" >> configmap.env


      - name: Create secrets file
        run: |
          cd Lago-K8S
          echo "POSTGRES_DB=${{ secrets.POSTGRES_DB }}" > secret.env
          echo "POSTGRES_PASSWORD=${{ secrets.POSTGRES_PASSWORD }}" >> secret.env
          echo "POSTGRES_USER=${{ secrets.POSTGRES_USER }}" >> secret.env
          echo "DATABASE_URL=postgresql://${{ secrets.POSTGRES_USER }}:${{ secrets.POSTGRES_PASSWORD }}@app-lago-postgres:5432/${{ secrets.POSTGRES_DB }}" >> secret.env
          echo "API_SECRET=${{ secrets.API_SECRET }}" >> secret.env

```
in `DATABASE_URL`, @`DB` has to match to postgres service's name and since we have `namePrefix: app-` in kustomization.yaml then it'll be `app-lago-postgres` (by default)

Same goes for `LAGO_REDIS_URL` : redis://redisservicename:6379 => `redis://app-lago-redis:6379` (by default)


It will create a secret/configmap based on these variables and can be modified only in Github repo's secret variables.

---

### **Secrets (Examples)**

**Postgres**

These are defined in `POSTGRES` folder.

Same goes for `POSTGRES_PASSWORD`,`POSTGRES_USER`

File : `./postgres/deployment.yaml`
```
  - name: POSTGRES_DB
    valueFrom:
      secretKeyRef:
        name: lago-secret
        key: POSTGRES_DB
```
---

**Api**

These are defined in `API` folder.

 `DATABASE_URL`, `API_SECRET`

File : `./api/deployment.yaml`, same goes for `./api/worker/deployment.yaml`, `./api/clock/deployment.yaml`

```
  - name: LAGO_RSA_PRIVATE_KEY
    valueFrom:
      secretKeyRef:
        name: lago-secret
        key: API_SECRET
```

```
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: lago-secret
        key: DATABASE_URL
```

### **ConfigMap (Examples)**

Also have to edit them in repo's secret variables, by default `LAGO_API_URL:http://localhost:3000` and `LAGO_FRONT_URL:http://localhost`

**Api**

```
  - name: LAGO_API_URL
    valueFrom:
      configMapKeyRef:
        name: lago-configmap
        key: LAGO_API_URL
```

```
  - name: LAGO_FRONT_URL
    valueFrom:
      configMapKeyRef:
        name: lago-configmap
        key: LAGO_FRONT_URL
```

---

Once everything is setup, To apply all the resources, run the following command:

`kustomize build . | kubectl apply -f - `

If you want to delete all the resources:

`kustomize build . | kubectl delete -f - `

---

For the RSA_PRIVATE_KEY

First you have to generate an RSA with :

```
openssl genpkey -algorithm RSA -out private_key.pem
```

```
openssl base64 -in private_key.pem -out private_key_base64.txt
```

----------------------------------------------------------------------------------------------------------------------------

For the ingress, you must install ALB : [@ALBIngress](https://github.com/TrouveTaVoie/eks-cluster)


once you add the certificate / domain name, make sure to edit the LAGO_API_URL and LAGO_FRONT_URL in your github repo's variables.

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
