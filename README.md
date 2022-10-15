#   USECASE-5 (Integration GIT,Maven,Jenkins , Docker, Tomcat, Ansible and kubernets )
 
 Please watch Usecase-2,Usecase-3  before come here
 
# What you will be learn this usecase :

As IT employees we work on many areas ( coding,support,infrastructure ) but while onboarding any new service we really are not aware how exactly code will deploy to production . There  are many possibilities  we can onboard our requirements to production .This use case will explain one of the ways to deploy our code to prod. 

This video  I have explained  how to troubleshoot the issues  also . After this recording was completed I had recorded another video without troubleshooting . But I feel People need to learn how to correct the mistakes also . So I preferred to upload this video . Keep Fail and learn more 

# Architecture
![Watch the image](/Ansible-kuber.png)

# Pre-request's:

1) Ansible -- Please chcek my previoues video --> https://lnkd.in/eaFVV3XX
2) Docker -- Please chcek my previoues video --> https://lnkd.in/eaFVV3XX
3) Jenkins -- Please chcek my previoues video -->  https://lnkd.in/eaZHhFbH
4) kubernetes -- Please chcek cloudnloud video's --> https://www.youtube.com/watch?v=md2BtnJYtt8
5) docker hub account -- > Signup  https://hub.docker.com/

# Steps


- step 1)  Check ssh connection between ansible server to kubernetes mater & update host details

- step 2 )create yml files in ansible for kubernets CI pipelines

- step 3) push image to docker hub and chcek
          3.1: Update Ansible server in jenkins
          3.2 : Create new maven job and run 
          3.3 : validate in docker hub

- step 4) create yml files in master servers for both deployment and service in /root

- step 5) create yml in  ansible server to run kubernetes yml servers

- step 6) create CD process using ansible and jenkins 

- step 7) link between CI and CD process

- step 8) Test the result

- step 9) Integrate to GIT



# Step 1)  Check ssh connection between ansible server to kubernetes mater & update host details
        
Login to ansible server then 
        
ssh -i id_rsa root@IP_addressof kubernetes_server
      
if connection not happen you need exchange keys between ansible server to kubernets serverr
        
login to ansible server 

          
          ssh-keygen
          
Press enter and main default everything

          ls -a
          cd .ssh/
          ssh-copy-id IPADDREE(second server ip address)

Update hosts file(kubernets ip) in aisible server

#  step 2 )create yml files in ansible for kubernets CI pipelines

mkdir /opt/kuber

vi createimage.yaml
```
---
- hosts: ansible-server
  become: true

  tasks:
  - name: create docker image using war file
    command: docker build -t tomcat-image:latest .
    args:
      chdir: /opt/kuber

  - name: create tag to image
    command: docker tag tomcat-image veejee2331/tomcat-image

  - name: push image on to dockerhub
    command: docker push veejee2331/tomcat-image

  - name: remove docker images form ansible server
    command: docker rmi tomcat-image:latest veejee2331/tomcat-image
    ignore_errors: yes
  
 ```
 




# step3.1 : Update Ansible server in jenkins

Install jenkins plugin  --> publish over ssh

manage jenkis --> configure systems --> publish over to ssh --> ssh servers --> add 

   name : ansiblemaster
  
  hostname: ipaddress (private bez of same vpc)
 
 username : ansadmin

 click advances session 
 
 password: asnadmin password 
 
 clieck test connection ( its show success or fail for our confirmation )

 
 Note: you have exchange the keys between jenkis and master server then only test connection will success


      
# step 3.2 :Create new maven job and run 

create job --> maven project -->

A) Source Code Management Repository : https://github.com/veejee2331/webapplication Branches to build : */main

B) Build Root POM: pom.xml Goals and options : clean install package

C) Post steps --> Add Post bulid step -- > select

select post steps --> select Send files or execute commands over SSH ( this is avaible once you insaltted publish over ssh plugin)
            update below info
	 
ssh server : name --> ansiablemaster ( automactally come bez we updated in congifg area on ealier steps)
transfer : source file--> webapp/target/*.war
removeprefix: webapp/target
Remote directory : //opt//Docker
	
	
once again 
	
select post steps --> select Send files or execute commands over SSH ( this is avaible once you insaltted publish over ssh plugin)
update below info

ssh server : name --> ansiablemaster ( automactally come bez we updated in congifg area on ealier steps)
exec command :  update below lines --> apply

```
cd /opt/docker
docker build -t tomcat_demo .
docker tag tomcat_demo  veejee2331/tomcat_demo
docker push veejee2331/tomcat_demo
docker rmi tomcat_demo veejee2331/tomcat_demo
```

Dockerfile ( /opt/docker location )

```
From tomcat:9-jre9
MAINTAINER "SREEKANTH"
COPY ./webapp.war /usr/local/tomcat/webapps/

```
# step 3.3 : Validate in Docker hub
How to chcek 

Login to https://hub.docker.com --> Repositories --> you can new image what we pushed 

#step4: create yml files in master servers for both deployment and service in /root

kubernetes-appdeploy.yml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
  labels:
    app: kube-ansible
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kube-ansible
  template:
    metadata:
      labels:
        app: kube-ansible
    spec:
      containers:
      - name: kube-ansible
        image: veejee2331/tomcat_demo
        ports:
        - containerPort: 80

```

kubernetes-appservice.yml

```
apiVersion: v1
kind: Service
metadata:
  name: app-service
  labels:
    app: kube-ansible
spec:
  type: NodePort
  selector:
   app: kube-ansible
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
```

#  step 5) create yml in  ansible server to run kubernetes yml servers

in anisble server /opt/kuber

vi ansible-deploy.yaml

```
---
- name: Create pods using deployment 
  hosts: kuber 
  
  user: root
 
  tasks: 
  - name: create a deployment
    command: kubectl apply -f kubernetes-appdeploy.yml
 
  - name: update deployment with new pods if image updated in docker hub
    command: kubectl rollout restart deployment app-deployment
    
 ```
 
 vi ansible-service.yml
 
 ```
 ---
- name: create service for deployment
  hosts: kuber
  
  user: root

  tasks:
  - name: create a service
    command: kubectl apply -f kubernetes-appservice.yml
    
 ```
 

# step 6) create CD process using ansible and jenkins

Go to jenkins --> create job --> freestyle project --> default --> post build action --> send build artifacts over SSH --> select ansible server --> exce command -->

ansible-playbook -i /opt/kuber/hosts /opt/kuber/ansible-deployment.yml; 
ansible-playbook -i /opt/kuber/hosts /opt/kuber/ansible-service.yaml; 

# step 7) link between CI and CD process

in step 3.2 we have created CI 

Go to CI job --> select the job --> configure --> post build action --> Select Build Other Projects --> project to build ( select cd job step6 job) --> apply --> save 

Now run(build ) the CI job 

# step 8) Test Result 

After both jobs completed CI and CD jobds then login to kubernets server --> chcek the deployment, pods  and service are created .

https://kubernetes-masterIPadress:nodeportnumber/uriname

# step 9 ) Integrate to GIT

Login to jenkins ci job --> configure --> enable poll scm --> give * * * * * ( every one min it will chcek the git area for updates )and save it 

Modify the some file on github location then automatcally CI/CD jobs will start.




