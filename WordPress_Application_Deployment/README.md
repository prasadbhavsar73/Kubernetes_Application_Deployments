Reference: https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/

# NFS

  ## Creating/Adjusting Folder /mysql & /html and editing /etc/exports

    $ cd nfs

    $ chmod u+x nfs.sh

    $ sudo ./nfs.sh
 
 Note: If you do not have NFS Server setup, you can make use of hostPath in PV. Refer: wordpress-mysql-pv-hostpath.yml


# K8s

  ## Creating a persistent volumes for Wordpress & MySQL

    $ cd k8s

    $ vim 00-wordpress-mysql-pv.yml

        replace "server: x.x.x.x" with NFS IP

    $ kubectl create -f 00-wordpress-mysql-pv.yml


  Verify a persistent volumes for Wordpress & MySQL

    $ kubectl get pv --show-labels

    NAME                           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM  STORAGECLASS   REASON    AGE       LABELS
      mysql-persistent-storage       10Gi       RWX            Retain           Available                                 43m       app=wordpress,tier=mysql
      wordpress-persistent-storage   10Gi       RWX            Retain           Available                                 43m       app=wordpress,tier=frontend


  ## Creating a persistent volume Claim MySQL

    $ kubectl create -f 01-mysql-pvc.yml    

  ## Creating a persistent volume Claim for Wordpress

    $ kubectl create -f 02-wordpress-pvc.yml

   Verfiy a persistent volume Claim for Wordpress &MySQL

    $ kubectl get pvc --show-labels

    NAME                           STATUS    VOLUME                         CAPACITY   ACCESS MODES   STORAGECLASS   AGE       LABELS
    mysql-persistent-storage       Bound     mysql-persistent-storage       10Gi       RWX                           42m       app=wordpress
    wordpress-persistent-storage   Bound     wordpress-persistent-storage   10Gi       RWX                           42m       app=wordpress

  Check the PV and see the STATUS and CLAIM

  the "STATUS" was changed from "Available" to "Bound"

  the "Claim" was changed from "NULL" to our new persistentVolumeClaim

    $ kubectl get pv --show-labels

    NAME                           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                                  STORAGECLASS   REASON    AGE       LABELS
    mysql-persistent-storage       10Gi       RWX            Retain           Bound     default/mysql-persistent-storage                                53m       app=wordpress,tier=mysql
    wordpress-persistent-storage   10Gi       RWX            Retain           Bound     default/wordpress-persistent-storage                            53m       app=wordpress,tier=frontend


  ## Creating Secret which used to our MySQL

     $ echo -n 'admin' | base64  

     $ kubectl create -f  03-secret.yml

  ## verify Secret which used to our MySQL

     $ kubectl get secret

        NAME                        TYPE                                  DATA      AGE
        mysql-pass                  Opaque                                1         19s


  ## Deploy MySQL with it's service "ClusterIP"

    $ kubectl create -f 04-mysql-deploy.yml

  Verfiy MySQL Service "wordpress-mysql", Port "3306", label "app=wordpress"

    $ kubectl get svc --show-labels

    NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE       LABELS
    wordpress-mysql   ClusterIP   None         <none>        3306/TCP   1m        app=wordpress

  Verify MySQL Deployment "mysql", desired "1", current "1"

    $ kubectl get deployment --show-labels

    NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       LABELS
    mysql       1         1         1            1           45m       app=wordpress

  Verify MySQL pod with label "app=wordpress,tier=mysql"

    $  kubectl get pods --selector=app=wordpress,tier=mysql

    NAME                     READY     STATUS    RESTARTS   AGE
    mysql-6d468bfbf7-g7hhh   1/1       Running   1          20m

  Verify the MySQL store the data on NFS

    $ ssh nfs-server

    $ ls -lh /mysql


  ## Deploy Wordpress with it's service "NodePort"

      $ kubectl create -f 05-wordpress-deploy.yml

  Verfiy Wordpress Service "wordpress", Port "80", NodePort "31004", label "app=wordpress"

      $ kubectl get svc --show-labels

      NAME              TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE       LABELS
      wordpress         NodePort    10.43.71.69   <none>        80:31004/TCP   21s       app=wordpress

  Verify Wordpress Deployment "wordpress", desired "2", current "2"

      $ kubectl get deployment --show-labels

      NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       LABELS
      wordpress   2         2         2            2           2m        app=wordpress

  Verify wordpress pod with label "app=wordpress,tier=frontend"

      $  kubectl get pods --selector=app=wordpress,tier=frontend

      NAME                         READY     STATUS    RESTARTS   AGE
      wordpress-5f85d888df-8s5tw   1/1       Running   0          11m
      wordpress-5f85d888df-l882q   1/1       Running   0          11m

  Verify the Wordpress store the data on NFS

      $ ssh nfs-server

      $ ls -lh /html


# Helm (Optional)

### Installing the Helm client

### Method 1: Installing Helm and Tiller

- Install helm with bash commands

   curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash

- Add a service account within a namespace to segregate tiller

    kubectl --namespace kube-system create sa tiller

- reate a cluster role binding for tiller
   
   kubectl create clusterrolebinding tiller \
       --clusterrole cluster-admin \
       --serviceaccount=kube-system:tiller

- Initialized helm within the tiller service account

   helm init --service-account tiller

- Updates the repos for Helm repo integration

   helm repo update

- Verify that helm is installed in the cluster.

   kubectl get deploy,svc tiller-deploy -n kube-system
   
### Method 2: Installing Helm and Tiller

- Download your desired version https://github.com/helm/helm/releases

- Unpack it (tar -zxvf helm-v2.0.0-linux-amd64.tgz)

- Find the helm binary in the unpacked directory, and move it to its desired destination (mv linux-amd64/helm /usr/local/bin/helm)

      $ wget https://storage.googleapis.com/kubernetes-helm/helm-v2.11.0-linux-amd64.tar.gz

      $ tar -xzvf helm-v2.11.0-linux-amd64.tar.gz

      $ sudo mv linux-amd64/helm /usr/local/bin/

      $ helm version

        Client: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}

### Installing Tiller

Tiller, the server portion of Helm, typically runs inside of your Kubernetes cluster.

    $ helm init --upgrade

    $ kubectl get deployment -n kube-system --selector=app=helm
      NAME            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
      tiller-deploy   1         1         1            1           1m

    $ kubectl get pods -n kube-system --selector=app=helm
      NAME                            READY     STATUS    RESTARTS   AGE
      tiller-deploy-759b9d56c-wcpxx   1/1       Running   0          2m


###  Create a new chart with the given name

      $ cd helm

      $ helm create wordpress

      $ helm install --name wordpress wordpress  --set nfs.server=x.x.x.x    #--set is optional.

      $ helm ls

        NAME     	REVISION	UPDATED                 	STATUS  	CHART          	NAMESPACE
        wordpress	1       	Wed Nov 14 00:58:25 2018	DEPLOYED	wordpress-0.1.0	default  

### Clean Up

      $ helm delete wordpress --purge
      
