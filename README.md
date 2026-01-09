 
# Flask App with MySQL Docker Setup

This is a simple Flask app that interacts with a MySQL database. The app allows users to submit messages, which are then stored in the database and displayed on the frontend.

## Prerequisites

Before you begin, make sure you have the following installed:

- Docker
- Git (optional, for cloning the repository)

## Setup

1. Clone this repository (if you haven't already):

   ```bash
   git clone https://github.com/your-username/your-repo-name.git
   ```

2. Navigate to the project directory:

   ```bash
   cd your-repo-name
   ```

3. Create a `.env` file in the project directory to store your MySQL environment variables:

   ```bash
   touch .env
   ```

4. Open the `.env` file and add your MySQL configuration:

   ```
   MYSQL_HOST=mysql
   MYSQL_USER=your_username
   MYSQL_PASSWORD=your_password
   MYSQL_DB=your_database
   ```

## Usage

1. Start the containers using Docker Compose:

   ```bash
   docker-compose up --build
   ```

2. Access the Flask app in your web browser:

   - Frontend: http://localhost
   - Backend: http://localhost:5000

3. Create the `messages` table in your MySQL database:

   - Use a MySQL client or tool (e.g., phpMyAdmin) to execute the following SQL commands:
   
     ```sql
     CREATE TABLE messages (
         id INT AUTO_INCREMENT PRIMARY KEY,
         message TEXT
     );
     ```

4. Interact with the app:

   - Visit http://localhost to see the frontend. You can submit new messages using the form.
   - Visit http://localhost:5000/insert_sql to insert a message directly into the `messages` table via an SQL query.

## Cleaning Up

To stop and remove the Docker containers, press `Ctrl+C` in the terminal where the containers are running, or use the following command:

```bash
docker-compose down
```

## To run this two-tier application using  without docker-compose

- First create a docker image from Dockerfile
```bash
docker build -t flaskapp .
```

- Now, make sure that you have created a network using following command
```bash
docker network create twotier
```

- Attach both the containers in the same network, so that they can communicate with each other

i) MySQL container 
```bash
docker run -d \
    --name mysql \
    -v mysql-data:/var/lib/mysql \
    --network=twotier \
    -e MYSQL_DATABASE=mydb \
    -e MYSQL_ROOT_PASSWORD=admin \
    -p 3306:3306 \
    mysql:5.7

```
ii) Backend container
```bash
docker run -d \
    --name flaskapp \
    --network=twotier \
    -e MYSQL_HOST=mysql \
    -e MYSQL_USER=root \
    -e MYSQL_PASSWORD=admin \
    -e MYSQL_DB=mydb \
    -p 5000:5000 \
    flaskapp:latest

```

Absolutely! I’ve drafted a **clear, step-by-step, beginner-friendly `README.md`** for your GitHub project. I wrote it as if a “baby” could follow it without missing anything, including all the steps you did on master and worker nodes, the Kubernetes setup, volumes, and port settings. I didn’t include the YAML files themselves, just instructions to apply them.

Here’s the `README.md`:

---

# SECOND HALF IN WHICH WE WILL USE K8S FOR DEPLOYMENT AND FAULT TOLRENECE.
---

## Prerequisites

* **Two Linux machines (EC2 instances)**:

  * One **master node**
  * One **worker node**
* **kubectl installed** on master
* **Kubernetes cluster initialized** (kubeadm init on master, kubeadm join on worker)
* **containerd installed** on both nodes
* **GitHub repo cloned** on master

---

## 1️. Project Folder Structure

1. On the master node:

```bash
mkdir ~/flaskapp-deployment
cd ~/flaskapp-deployment
mkdir k8s
```

2. Place the following files in the `k8s/` folder:

* `two-tier-app-pod.yaml`
* `two-tier-app-deployment.yaml`
* `two-tier-app-service.yaml`
* `mysql-deployment.yaml`
* `mysql-pv.yml`
* `mysql-pvc.yml`
* `mysql-service.yaml`

3. Other project files in root:

* `Dockerfile`
* `app.py`
* `docker-compose.yaml`
* `requirements.txt`
* `message.sql`
* `templates/index.html`

Your tree should look like this:

```
flaskapp-deployment/
├── Dockerfile
├── README.md
├── app.py
├── docker-compose.yaml
├── requirements.txt
├── message.sql
├── templates/
│   └── index.html
└── k8s/
    ├── mysql-deployment.yaml
    ├── mysql-pv.yml
    ├── mysql-pvc.yml
    ├── mysql-service.yaml
    ├── two-tier-app-deployment.yaml
    ├── two-tier-app-pod.yaml
    └── two-tier-app-service.yaml
```

---

## 2️. Worker Node Preparation

On the **worker node**, create the directory for MySQL data:

```bash
sudo mkdir -p /var/lib/mysql-data
sudo chmod 777 /var/lib/mysql-data
```

> ✅ This path is used in the PersistentVolume for MySQL.

---

## 3️. Deploy Persistent Volume (PV) & Persistent Volume Claim (PVC)

On the **master node**:

```bash
kubectl apply -f k8s/mysql-pv.yml
kubectl apply -f k8s/mysql-pvc.yml
kubectl get pv
kubectl get pvc
```

> Make sure `STATUS` shows `Available` for PV and `Bound` for PVC.

---

## 4️. Deploy MySQL

1. Apply the MySQL deployment:

```bash
kubectl apply -f k8s/mysql-deployment.yaml
```

2. Apply the MySQL service:

```bash
kubectl apply -f k8s/mysql-service.yaml
```

3. Check the pod:

```bash
kubectl get pods -l app=mysql -o wide
```

5. **Find the MySQL service cluster IP** (to connect from your Flask app):

```bash
kubectl get svc mysql
```

> Example output:

```
NAME    TYPE        CLUSTER-IP       PORT(S)
mysql   ClusterIP   10.107.164.200   3306/TCP
```

> Use this `CLUSTER-IP` in your Flask app deployment as `MYSQL_HOST`.

---

## 5️. Update Flask App Deployment

In `two-tier-app-deployment.yaml`, set environment variables:

```yaml
env:
  - name: MYSQL_HOST
    value: "10.107.164.200"    # <-- Cluster IP from mysql service
  - name: MYSQL_USER
    value: "admin"
  - name: MYSQL_PASSWORD
    value: "admin"
  - name: MYSQL_DB
    value: "mydb"
```

---

## 6️. Deploy Flask App

1. Apply the deployment:

```bash
kubectl apply -f k8s/two-tier-app-deployment.yaml
```

2. Apply the service:

```bash
kubectl apply -f k8s/two-tier-app-service.yaml
```

3. Check the pods:

```bash
kubectl get pods -l app=two-tier-app
```

---

## 7️. Expose NodePort

1. In AWS Management Console (or your cloud provider), allow **inbound traffic** to:

* **Flask App NodePort**: `30007`
* **MySQL**: Usually **3306**, if you want external access (optional, for cluster internal access only, skip this)

> ✅ The NodePort is set in `two-tier-app-service.yaml`:

```
ports:
  - port: 5000
    targetPort: 5000
    nodePort: 30007
```

---

## 8️. Access Your App

1. Get the public IP of your worker node:

```bash
curl http://<worker-public-ip>:30007
```

2. You should see the Flask app running.
3. To check MySQL data:

```bash
sudo crictl ps           # List containers
sudo crictl exec -it <mysql-container-id> /bin/bash
mysql -u admin -p
```

---

## 9️. Common Commands

* Check all pods:

```bash
kubectl get pods -o wide
```

* Check logs:

```bash
kubectl logs <pod-name>
```

* Restart deployment:

```bash
kubectl rollout restart deployment <deployment-name>
```

* Delete deployment:

```bash
kubectl delete -f <deployment-yaml>
```

---

##  Notes

* **Do not hardcode pod IPs** — use the **service name or Cluster IP**.
* When stopping/restarting EC2 instances, **public IPs may change**. Use Elastic IPs if you want a permanent URL.
* Persistent volume ensures MySQL data is **not lost** when pods restart.
* Cluster pods communicate internally via **service DNS names**; no need to update IPs manually.



