# notepad App on k8s

![k8s logo](https://cncf-branding.netlify.app/img/projects/kubernetes/horizontal/color/kubernetes-horizontal-color.png)

[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

This repo contains k8s deployment yml files for deploying simple notepad spring boot app on k8s cluster.

# About the app
  - App repo: https://github.com/jorgeacetozi/notepad
  - this simple application is used for demo purposes only
  - it's a simple spring application that uses mysql DB to store notes and it has three endpoints
    - /health endpoint: to check if application is running 
    - get endpoint: to get the already created notes in mysql DB
    - post endpoint: to create new notepad

# Repo's main focus
this repo will only focus on deploying this app on k8s cluster. if you follow along with this documented work, you should end up having two pods of running notepad app that persist their data on one mysql DB pod.

### k8s environment setup
  - Cluster
        - one control plane
        - two worker nodes
  - components versions
        - kubelet=1.19.1-00, kubeadm=1.19.1-00, kubectl=1.19.1-00
        - Ubuntu 18.04 LTS (Bionic Beaver)
  - network addon: Calico

I used my linux academy account to create this cluster as shown below

![Cluster](https://user-images.githubusercontent.com/17851915/104832522-4aea2980-589a-11eb-8d00-32d259ea5f60.png)
- host names
        - control plane:  KarimOthman2c.mylabserver.com 
        - worker1:  KarimOthman3c.mylabserver.com 
        - worker2:  KarimOthman1c.mylabserver.com 

### Building image steps
1- clone code repo into your control plane machine
2- install maven  
3- build code

    $ git clone https://github.com/jorgeacetozi/notepad.git
    $ sudo apt install maven
    $ sudo mvn clean install package -Dmaven.test.skip=true
4- login to docker hub registry
5- Create this Secret, naming it regcred
6- head to dockerfile directory and build image

    $ docker login
    $ kubectl create secret docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
    $ sudo docker build -t karimothman/notepad:latest .

7- show built image

    $sudo docker images
![image](https://user-images.githubusercontent.com/17851915/104832962-15dfd600-589e-11eb-8fe1-14917d767c33.png)

8- push image to docker hub
    
    $sudo docker push karimothman/notepad
    
### Create mysql setup
1- define my-sql storage class yml file "mysql-sc.yml" and make sure to set "allowVolumeExpansion" with true for future expansion if needed
2- define persistence volume yml file "mysql-pv.yml" with persistentVolumeReclaimPolicy= retain in to keep data in it. Also, using host path isn't the best practice in multi nodes env --- accordingly, I'll restrict mysql deployment to one node with label mysql-persist-node
3- label worker2 "karimothman1c.mylabserver.com" node with mysql-persist-node

    $kubectl label nodes "karimothman1c.mylabserver.com"  mysql-persist-node=true
4- define persistence volume claim "mysql-pvc.yml" to index pv
5- create the previous defined sc and pv and pvc
    
    $kubectl create -f mysql-sc.yml
    $kubectl create -f mysql-pv.yml
    $kubectl create -f mysql-pvc.yml
6- create secret for mysql password

    $kubectl create secret generic mysecrets --from-literal=MysqlPass=root
7- define mysql deployment yml file "mysql-deployment.yml".
- this file should
    - deploy one replica of mysql:5.7 image
    - (as mentioned before) use nodeSelector and restrict the deployment to "mysql-persist-node" nodes
    - reference "MysqlPass" secret created before as MYSQL_ROOT_PASSWORD

8- create the defined “mysql-deployment.yml” file

    $kubectl create -f mysql-deployment.yml 
9- define clusterIP service "mysql-service.yml" to expose mysql pods in cluster
10- create mysql-service.yml

    $kubectl create -f mysql-service.yml

### Create notepad app setup
1- define notepad deployment file "notepad-deployment.yml".
- this file should 
    - use pre-created docker hub secret (imagePullSecrets --> regcred) to pull "karimothman/notepad:latest" image
    - run two replicas of it on any k8s node with no restriction as applied on mysql
    - have readinessProbe on path: /health
    - reference pre-created clusterIP service "mysql-service" on ENV_TEST_MYSQL_HOST environment variable
    
2- create the defined "notepad-deployment.yml" file

    $kubectl create -f notepad-deployment.yml
3- define "notepad-nodeportservice.yml" file to expose notepad pods through NodePort service (simplest technique)
4- create the defined "notepad-nodeportservice.yml"

    $kubectl create -f notepad-nodeportservice.yml
### Trouble Shooting Guide
1- Check if pods are up

    $kubectl get pods -o wide
![image](https://user-images.githubusercontent.com/17851915/104834136-33b13900-58a6-11eb-9d70-5576ad09e6e8.png)
2- you may find crash loop on notepad pods; check pod logs for more details

    $kubectl logs -p notepad-78f544b4b5-8kbbt
![image](https://user-images.githubusercontent.com/17851915/104834217-af12ea80-58a6-11eb-8b56-70dc3d940933.png)

3- previous error indicates that mysql ClusterIP service isn't accessible.

- ping mysql service from inside notepad pod

    
    $kubectl exec notepad-78f544b4b5-8kbbt  -- ping mysql

- make sure that you are defining "ENV_TEST_MYSQL_HOST" environment variable in a right way

### Setup illustration
![image](https://user-images.githubusercontent.com/17851915/104834630-66106580-58a9-11eb-8591-0c513cde3847.png)
