# digitalocean-kubernetes-sql-cluster

Our goal is to deploy a scalable SQL database cluster. Database cluster must be redundant and scalable.
The following guide contains instructions for deploying MySQL database cluster using official MySQL Operator for Kubernetes.

Database cluster will be deployed on managed Kubernetes cluster on Digital Ocean.

Prerequisites:
- Digital Ocean account (https://www.digitalocean.com/)
- kubectl (https://kubernetes.io/docs/tasks/tools/)
- helm (https://helm.sh/)
- Text editor
- Client for MySQL database (I will be using DataGrip by JetBrains).

## 1. Creating Kubernetes cluster

First you need to create a Kubernetes cluster. I recommend using the official Digital Ocean documentation: https://docs.digitalocean.com/products/kubernetes/quickstart/

Once the cluster is created you can download the cluster configuration file. In my case the name of the configuration file was k8s-mysql-cluster-kubeconfig.yaml which I downloaded to my local machine into my working directory for this project.

Then we can set the KUBECONFIG environment variable and test connection to our cluster by displaying the list of nodes:
![IMAGE1.png](images/IMAGE1.png)

## 2. Install mysql operator

We will clone the mysql operator code from offical github repository and install it into our cluster using the helm util.
```
git clone https://github.com/mysql/mysql-operator.git
cd mysql-operator
helm install mysql-operator helm/mysql-operator --namespace mysql-operator --create-namespace
```
![IMAGE2.png](images/IMAGE2.png)

## 3. Set up the cluster

First we create a namespace for our cluster. 
```
kubectl create namespace sql-cluster
```
![IMAGE3.png](images/IMAGE3.png)

Then we create a secret to hold our database root password:
```
kubectl create secret generic mypwds --from-literal=rootUser=root --from-literal=rootHost=%         --from-literal=rootPassword="my secret password" --namespace sql-cluster
```
![IMAGE4.png](images/IMAGE4.png)

Then we prepare and apply our mysql cluster definition file (cluster.yaml). We will initially set it to 3 instances:
```yaml
apiVersion: mysql.oracle.com/v2alpha1
kind: InnoDBCluster
metadata:
  name: mycluster
  namespace: sql-cluster
spec:
  secretName: mypwds
  instances: 3
  router:
    instances: 1
```    

Apply it with:
```
kubectl apply -f cluster.yaml
```

Observe provisioning proces with:
```
kubectl get innodbcluster --watch  --namespace sql-cluster
```

![IMAGE5.png](images/IMAGE5.png)

It will take about 3 minutes (depending your cluster resources) for all 3 instances to get online.

You can also observe the state of your cluster in your Kubernetes Dashboard which is accessible through the Digital Ocean Kubernetes management panel:
![IMAGE6.png](images/IMAGE6.png)


## 4. Connect to database cluster and test it

Our database cluster is current not accessible outside of our Kubernetes cluster. To access it from our local machine we need to use the kubectl port-forward command. We can see that database is exposed on port 6446 by printing information about the service:

```
kubectl get service mycluster  --namespace sql-cluster
kubectl describe service mycluster  --namespace sql-cluster
kubectl port-forward service/mycluster mysql  --namespace sql-cluster
```
![IMAGE7.png](images/IMAGE7.png)

Now we can access the database using any MySQL client by connecting to root@127.0.0.1:6446 for example in DataGrip client:
![IMAGE8.png](images/IMAGE8.png)

![IMAGE9.png](images/IMAGE9.png)

We test the database by creating a simple table on new schema:
```sql
create schema test;
use test;

CREATE TABLE Users (
id INT(6) UNSIGNED AUTO_INCREMENT PRIMARY KEY,
firstname VARCHAR(30) NOT NULL,
lastname VARCHAR(30) NOT NULL,
email VARCHAR(50),
reg_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
)

INSERT INTO Users (firstname, lastname ) values ("John", "Doe");

SELECT * from Users;
```

![IMAGE10.png](images/IMAGE10.png)

## 5. Scale the database cluster

If we need to scale database cluster we can simply change the number of instances in our cluster.yml file and apply it again:

```yaml
apiVersion: mysql.oracle.com/v2alpha1
kind: InnoDBCluster
metadata:
  name: mycluster
  namespace: sql-cluster
spec:
  secretName: mypwds
  instances: 4
  router:
    instances: 1
```

Apply changes with:
```
kubectl apply -f cluster.yaml
```

Observe scaling proces with:
```
kubectl get innodbcluster --watch  --namespace sql-cluster
```

![IMAGE11.png](images/IMAGE11.png)

Check instances in database metadata tables:
![IMAGE12.png](images/IMAGE12.png)

## 6. Test the resiliancy by deleting one instance

Connect to database again and check which host is currently being used:
```sql
SHOW VARIABLES WHERE Variable_name = 'hostname'
```
![IMAGE13.png](images/IMAGE13.png)

Delete the pod of this mysql host:
```
kubectl delete -n sql-cluster pod mycluster-0
```
![IMAGE14.png](images/IMAGE14.png)

Check the current host again in the same database session. We see our session has been moved to a new host and our data is still avaiable:

![IMAGE15.png](images/IMAGE15.png)


## 7. Conclusion

This concludes the basic setup and testing of our MySQL database cluster.
You can now use this cluster as database backend for your applications in the Kubernetes cluster.