# Understanding the Project Files: A Beginner's Guide to Kubernetes

This guide will walk you through the core files of a Kubernetes WordPress deployment project. We'll break down the Kubernetes manifests into simple, easy-to-understand concepts. This is designed for those new to Kubernetes.

**Kubernetes Manifests: Defining Your Application**
Kubernetes uses YAML files, called "manifests," to define the desired state of your application. Think of these as blueprints that tell Kubernetes exactly what to build and how to run it. Our project uses three manifest files located in the `k8s/` directory.

`01-mysql-deployment.yml`: **Setting Up the Database**
This file is responsible for creating a persistent MySQL database for our WordPress site. Let's look at its three main parts:

1. `PersistentVolumeClaim` **(PVC)**

* **What it is:** A PVC is a request for storage. Since containers can be deleted and recreated, we need a way to store data permanently. The PVC, named mysql-pvc, requests 1 gigabyte (1GB) of storage.
* **Why it's needed:** This ensures that your MySQL data (like blog posts and user comments) persists even if the MySQL container (Pod) crashes or is restarted.
* **Key concept:** The `accessModes: - ReadWriteOnce` setting means that this storage can only be attached to and used by a single node at a time, which is suitable for a database.

2. `Deployment`
* **What it is:** A Deployment manages the lifecycle of your application's containers. It ensures that a specified number of replicas (copies) of your container are always running. In this case, we have `replicas: 1`. It means, one container should be always running.
* **How it works:** It uses the official `mysql:5.7` Docker image to create the MySQL container.
* **Environment Variables (`env`):** These are used to configure the MySQL container. Instead of hardcoding sensitive information, we use `valueFrom` to securely load the database name, user, and passwords from a Kubernetes Secret named `mysql-secrets`. These secrets will be created later.
* **`volumeMounts`:** This connects the `PersistentVolumeClaim` we created to a specific path inside the container (`/var/lib/mysql`), which is where MySQL stores its data.

3.`Service`
* **What it is:** A Service provides a stable network endpoint (like a DNS name and IP address) for accessing your application. Containers can be ephemeral, but a Service's address remains constant. This means that containers can be accessed even if they receive new ip adresses.
* **How it works:** This Service, named `mysql-service`, exposes the MySQL database on port `3306`.
* **Why it's needed:** It allows our WordPress application to connect to the MySQL database using a consistent name (`mysql-service`) without needing to know the specific IP address of the MySQL container.


`02-wordpress-deployment.yml`: **Deploying the WordPress Application**

This file sets up the WordPress site itself. It's similar in structure to the MySQL file.

1. `Deployment`
* **What it is:** This Deployment manages the WordPress containers, ensuring that one replica is always running.
* **How it works:**  It uses the latest official `wordpress` Docker image.
* **Environment Variables (`env`):** This is where the magic happens for connecting WordPress to its database.
    * `WORDPRESS_DB_HOST`:  Is set to `mysql-service:3306`, telling WordPress to connect to the MySQL Service we created earlier.

    * `WORDPRESS_DB_USER`, `WORDPRESS_DB_PASSWORD`, and `WORDPRESS_DB_NAME`: These values are also loaded from our mysql-secrets Kubernetes Secret, ensuring WordPress uses the correct credentials to access the database. 

2. `Service`
* **What it is:** This Service, named `wordpress-service`, exposes the WordPress application on port `80` within the cluster.
* **Why it's needed:** While this service allows other applications inside the Kubernetes cluster to communicate with WordPress, it doesn't make the site accessible to the public internet. For that, we need an Ingress (it is similar to a reverse proxy).


`03-wordpress-ingress.yml`: Exposing WordPress to the World

This file is the final piece of the puzzle, allowing external traffic to reach our WordPress site.

* `Ingress`

    * **What it is:** An Ingress is a Kubernetes object that manages external access to the services in a cluster, typically HTTP and HTTPS. For this project, we will use http.
    * **How it works:** This Ingress rule, named `wordpress-ingress`, directs all incoming traffic (`path: /`) to the `wordpress-service` on port 80.
    * **Annotations:** The `kubernetes.io/ingress.class: "traefik"` annotation tells K8s's built-in nginix Ingress controller to handle this rule.


## Run this project in your Local MiniKube Cluster.

If you want to run this project locally, just follow the below steps:

1. **Create the MySQL secret**  
   Choose one method:

   **a. Using `kubectl`**  
   ```bash
   kubectl create secret generic mysql-secrets \
     --from-literal=MYSQL_ROOT_PASSWORD=root-password \
     --from-literal=MYSQL_PASSWORD=user-password

     ```

   **b. Using a YAML file**

    Save this as mysql-secret.yaml:
    ```bash
    apiVersion: v1
    kind: Secret
    metadata:
        name: mysql-secrets
    type: Opaque
    stringData:
        MYSQL_ROOT_PASSWORD: root-password
        MYSQL_PASSWORD:      user-password
    ```

    Then run:

    `kubectl apply -f mysql-secret.yaml`

    **c. Verify if the secrets were created successfully**

    ```bash
    kubectl get secrets
    ```


2. **Since we are using Ingress, we need to configure the ingress addon if not already enabled.**

    ```bash
    minikube addons enable ingress
    ```

    Wait a few seconds for the `ingress-nginx-controller` Pod to be ready:

    ```bash
    kubectl get pods -n ingress-nginx
    ```

    After some minutes, the ingress-nginx-controller should be in running state. The other Ingress-nginx-admision-xxx should be in complete status.

3. **Deploy all resources**
    `k8s` is the folder that contains all the kubernetes Manifests.

    ```bash
    kubectl apply -f k8s
    ```

4. **Verify if pods are running successfully**

    ```bash
    kubectl get pods
    ```

5. **Accessing the WordPress application:**


    **a. Get Minikube IP:**

    ```bash
    minikube ip
    ```

    **b. Using the Minikube ip to access the application from your browser:**

    In your browser, using the minikube ip, use the below url:

    `http://<yourMiniKubeIP>/wp-admin/`


## Deleting your kubernetes cluster project.


### Option 1: Delete All Resources from Your YAML Files**

```bash
kubectl delete -f k8s/
```

This deletes:

- WordPress Deployment
- MySQL Deployment
- Services
- Ingress
- PVCs (unless protected by `persistentVolumeReclaimPolicy`)

**To confirm all resources are gone:**

```bash
kubectl get all
kubectl get ingress
kubectl get pvc
```


### Option 2: Delete Everything and Reset Minikube (optional)


If you're done experimenting and want a fresh Mini Kube start:

```bash
minikube delete
```

This stops and deletes the entire Minikube VM and its resources.

To start again later:

```bash
minikube start
```