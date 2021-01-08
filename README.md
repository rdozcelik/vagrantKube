# vagrantKube

Automated deployment PoC for Jenkins on Kubernetes

[TOC]

## Prerequisite

- VirtualBox
- Vagrant
- Vagrant Plugin "vagrant-disksize"

## Deploy Everything

cd into "vagrantKube" directory and run

```bash
$ vagrant up
```

Files are mostly self explanatory, general flow is like:

* Vagrantfile is the entrypoint
* Create worker1, worker2 and master VirtualBox VMs in order
* Provision phase starts after master VM is deployed
* Vagrant "ansible_local" module is used < master is the Ansible Server >
* Then vagrant provisions with ansible playbooks:
  * ansible/master.yaml
  * ansible/workers.yaml
  * ansible/config-on-all.yaml
  * ansible/config-on-master.yaml
* Playbooks handles everything from Docker installation to PV-PVC creation and Jenkins/Nexus Deployments

After deploy is done you can ssh into master, worker1 and worker2 VMs with ssh/id_rsa private key.

SSH into master and check the general status:

```bash
vagrant@master:~$ kubectl get nodes
NAME      STATUS   ROLES                  AGE   VERSION
master    Ready    control-plane,master   41m   v1.20.1
worker1   Ready    <none>                 39m   v1.20.1
worker2   Ready    <none>                 37m   v1.20.1

vagrant@master:~$ kubectl get service -n jenkins
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
glusterfs-cluster   ClusterIP   10.108.11.197   <none>        1/TCP             25m
jenkins             NodePort    10.108.157.66   <none>        8080:32000/TCP    25m
nexus               NodePort    10.98.5.252     <none>        8081:32001/TCP    25m
nexus-registry      NodePort    10.98.214.27    <none>        32032:32032/TCP   25m

vagrant@master:~$ kubectl get pods -n jenkins
NAME                     READY   STATUS    RESTARTS   AGE
jenkins-9f678479-tvgmw   1/1     Running   0          25m
nexus-547f4d7d74-nd8gn   1/1     Running   0          25m
```

Check GlusterFS cluster status:

```bash
vagrant@master:~$ sudo gluster peer status
Number of Peers: 2

Hostname: 192.168.10.11
Uuid: 4edcd294-0a45-4fd9-a21c-ee7a86d4a8d3
State: Peer in Cluster (Connected)

Hostname: 192.168.10.12
Uuid: 6714e5b0-8e6e-4c24-974d-bea929e876b5
State: Peer in Cluster (Connected)

vagrant@master:~$ sudo gluster volume info
 
Volume Name: jenkinsVol
Type: Distribute
Volume ID: e81afd41-403d-4053-8921-e2b598c40279
Status: Started
Snapshot Count: 0
Number of Bricks: 3
Transport-type: tcp
Bricks:
Brick1: 192.168.10.10:/data/jenkinsVol
Brick2: 192.168.10.11:/data/jenkinsVol
Brick3: 192.168.10.12:/data/jenkinsVol
Options Reconfigured:
transport.address-family: inet
storage.fips-mode-rchecksum: on
nfs.disable: on
 
Volume Name: nexusVol
Type: Distribute
Volume ID: c65c3e54-37cf-483f-b81e-7018d79f93bc
Status: Started
Snapshot Count: 0
Number of Bricks: 3
Transport-type: tcp
Bricks:
Brick1: 192.168.10.10:/data/nexusVol
Brick2: 192.168.10.11:/data/nexusVol
Brick3: 192.168.10.12:/data/nexusVol
Options Reconfigured:
transport.address-family: inet
storage.fips-mode-rchecksum: on
nfs.disable: on
```

## Jenkins

1. Get initial admin password from logs

   ```bash
   vagrant@master:~$ kubectl get pods -n jenkins
   vagrant@master:~$ kubectl logs <jenkins_pod_name> -n jenkins
   ```

2. Go to the Jenkins web-ui http://192.168.10.10:32000/login

3. Enter admin password

4. Select Install Suggested Plugins

5. Create an admin user and password

6. Re-login with `user: admin` `password: <admin_password>`

7. Install "Kubernetes" and "workflow-aggregator" plugins

## Nexus

1. Go to the Nexus web-ui http://192.168.10.10:32001

2. Get the admin password with the following command:

   ```bash
   vagrant@master:~$ cat /data/nexusVol/admin.password | xargs echo 
   ```

3. Login with admin user and credential

4. Enable anonymous access 

5. Administration -> Repositories -> Create Blob store -> docker-hubby-blob

6. Administration -> Repositories -> Create Repository: docker (hosted) -> docker-hubby

![image-20210108112920815](How-to%20DObase.assets/image-20210108112920815.png)

![image-20210108113118764](How-to%20DObase.assets/image-20210108113118764.png)

7. Administration -> Security Realms -> Activate "Docker Bearer Token Realm"

8. Login with nexus admin user and password on each node

   ```bash
   sudo docker login -u admin 192.168.10.10:32032
   ```

9. Create new tag to make a test

   ```bash
   sudo docker image tag nginx:1.14.2 192.168.10.10:32032/repository/docker-hubby/nginx:latest
   ```

10. Push the image

    ```bash
    sudo docker push 192.168.10.10:32032/repository/docker-hubby/nginx:latest
    ```

11. Success, nexus container registry up and running

## Jenkins & Kubernetes Integration

Get Kubernetes Service Account Credentials for jenkins service-account, which is created during the automated deployment. Follow the reference article and add Kubernetes credentials to the Jenkins via web-ui.

> Ref: https://support.cloudbees.com/hc/en-us/articles/360038636511-Kubernetes-Plugin-Authenticate-with-a-ServiceAccount-to-a-remote-cluster

```bash
$ kubectl get secret $(kubectl get sa jenkins -n jenkins -o jsonpath={.secrets[0].name}) -n jenkins -o jsonpath={.data.token} | base64 --decode
```

```bash
$ kubectl get secret $(kubectl get sa jenkins -n jenkins -o jsonpath={.secrets[0].name}) -n jenkins -o jsonpath={.data.'ca\.crt'} | base64 --decode
```

## Alternative Jenkins installation with Helm v3 charts

### Configure Helm

Once Helm is installed and set up properly, add the Jenkins repo as follows:

```bash
helm repo add jenkinsci https://charts.jenkins.io
helm repo update
```

The helm charts in the Jenkins repo can be listed with the command:

```bash
helm search repo jenkinsci
```

Use created jenkins service-account for Kubernetes.

* To enable persistence, we will create an override file and pass it as an argument to the Helm CLI. Paste the content from [raw.githubusercontent.com/jenkinsci/helm-charts/main/charts/jenkins/values.yaml](https://raw.githubusercontent.com/jenkinsci/helm-charts/main/charts/jenkins/values.yaml) into a YAML formatted file called `jenkins-values.yaml`.

### Jenkins Helm Chart

The `jenkins-values.yaml` is used as a template to provide values that are necessary for setup.

1. Open the `jenkins-values.yaml` file in your favorite text editor and modify the following:

   - nodePort: Because we are using minikube we need to use NodePort as service type. Only cloud providers offer load balancers. We define port 32000 as port.

     ```diff
     -  serviceType: ClusterIP
     +  serviceType: NodePort
     +  nodePort: 32000
     ```

   - storageClass:

     ```
     storageClass: jenkins-pv
     ```

   - serviceAccount: the serviceAccount section of the jenkins-values.yaml file should look like this:

     ```
     serviceAccount:
     create: false
     # Service account name is autogenerated by default
     name: jenkins
     annotations: {}
     ```

     ```
     Where `name: jenkins` refers to the serviceAccount created for jenkins.
     ```

   - Persistent volume:

     ```diff
     persistence:
       enabled: true
       existingClaim: jenkins-volume-claim
       storageClass: jenkins-pv
       annotations: {}
       accessMode: "ReadWriteOnce"
       size: "10Gi"
       volumes:
       - name: jenkins-pv
         emptyDir: {}
       mounts:
       - mountPath: /data
         name: jenkins-pv
         readOnly: false
     ```

   - We can also define which plugins we want to install on our Jenkins. We use some default plugins like git and the pipeline plugin.

2. Now you can install Jenkins by running the `helm install` command and passing it the following arguments:

   - The name of the release `jenkins`

   - The -f flag with the YAML file with overrides `jenkins-values.yaml`

   - The name of the chart `jenkinsci/jenkins`

   - The `-n` flag with the name of your namespace `jenkins`

     ```bash
     $ chart=jenkinsci/jenkins
     $ helm install jenkins -n jenkins -f jenkins-values.yaml $chart
     ```

     This outputs something similar to the following:

     ```
     NAME: jenkins
     LAST DEPLOYED: Wed Sep 16 11:13:10 2020
     NAMESPACE: jenkins
     STATUS: deployed
     REVISION: 1
     
     ```

     Follow Kubernetes event logs in parallel just in case:

     ```bash
     kubectl get events -n jenkins --watch
     ```

     If you get any crash on jenkins init, check with:

     ```bash
     kubectl logs jenkins-0 -c init -n jenkins
     ```

     ```bash
     vagrant@master:~$ kubectl get service -n jenkins
     NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
     jenkins         NodePort    10.99.19.103    <none>        8080:32000/TCP   86s
     jenkins-agent   ClusterIP   10.105.227.71   <none>        50000/TCP        86s
     
     vagrant@master:~$ kubectl get pods -n jenkins
     ```

* Get your 'admin' user password by running:

```bash
jsonpath="{.data.jenkins-admin-password}"
secret=$(kubectl get secret -n jenkins jenkins -o jsonpath=$jsonpath)
echo $(echo $secret | base64 --decode)
```

* Get the Jenkins URL to visit by running these commands in the same shell:

```bash
jsonpath="{.spec.ports[0].nodePort}"
NODE_PORT=$(kubectl get -n jenkins -o jsonpath=$jsonpath services jenkins)
jsonpath="{.items[0].status.addresses[0].address}"
NODE_IP=$(kubectl get nodes -n jenkins -o jsonpath=$jsonpath)
echo http://$NODE_IP:$NODE_PORT/login
```

Disable csrf, because it throws no valid crumb error and prevents login on Jenkins web-ui:

Create $JENKINS_HOME/init.groovy :

```groovy
import jenkins.model.Jenkins
def instance = Jenkins.instance
instance.setCrumbIssuer(null)
```

> **This method is abandoned as it causes many troubles. Will be revisited later.**

## Deploy the image on the Kubernetes Cluster with Helm

#TODO

1. Simple-maven-app is forked into https://github.com/rdozcelik/simple-java-maven-app
2. Will be deployed by following Jenkins tutorial

## Design a Pipeline

#TODO

# Screenshots

#TODO
