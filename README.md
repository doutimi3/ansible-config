# CI/CD pipeline for a PHP-based application using Jenkins, Ansible, Nexus Repository, and Sonarqube

In this project, I will walk you through the entire concept of CI/CD from an application perspective and present a hands-on guide to enable you to set up a CI/CD pipeline to deploy an application. This project has some initial theoretical concepts that must be well understood before moving on to the practical part. To successfully implement this project, it is crucial to grasp the importance of the entire CI/CD process, the roles of each tool, and success metrics. Refer to my previous article, "(Continuous Integration and Continuous Development (CI/CD) and its Importance in DevOps)[https://medium.com/@angalabiridortimiariyemaxwell/continuous-integration-and-continuous-development-ci-cd-and-its-importance-in-devops-d7b7b6492177]" to read up on the theoretical concepts needed to successfully understand the implementation of this project.
In this project, I will be simulating an end-to-end continuous integration and delivery pipeline to deploy a TODO web application. This application is built using PHP, which is an interpreted scripting language. This means it can be deployed directly onto a server and will work without compiling the code to a machine language (which is the case for compiled languages like Java,.NET, etc.).
The problem with that approach is that it would be difficult to package and version the software for different releases. And so, in this project, I will be using a different approach for releases. Rather than downloading the source code directly from Git, I will be using the Ansible uri module.

**Project Architecture**

The architectural diagram of the project is shown below:

![](./img/CI_CD-Pipeline-For-PHP-ToDo-Application.png)

Simulating all the various environments, from dev/ci to production, for this project will require a considerable number of servers. However, it is advisable to only create the servers necessary for the environment currently being worked on. For instance, when deploying for development, do not create servers for integration, pentest, or production yet.
To get started, we will initially focus on these environments.
Ci
Dev
Pentest

The SIT (System Integration Testing) and UAT (User Acceptance Testing) environments are essentially the webservers holding the application so they don't need additional installation or configuration. However, Pentest (Penetration Testing) requires specific configurations and additional tools as it is primarily used for security-related tests. In some cases, it may also be used for performance and load testing. Alternatively, it may be a separate environment on its own, depending on the team and company decisions.

In this project, Nginx will be used to serve as a reverse proxy for our sites and tools. Each environment setup is represented in the below table and diagrams.

![](./img/Table%20for%20Environment-setup.png)

ci

![](./img/CI-EnvironmentDiagram.png)

Others

![](./img/OtherEnvironments_Diagram.png)

__DNS requirements__

Make DNS entries to create a subdomain for each environment. Assuming your main domain is todoapp.com. This can be done on Godaddy, AWS Route53, or any other domain management provider.
You should have a subdomain list like this:

![](./img/domains.png)

__STEP 1: Setup Ansible Configurations__

1. launch a t2-medium RHEL server to run jenkins and ansible, ssh into this instance and initialize a new git repo called "ansible-config"
2. In the ansible-config directory create an inventory directory with inventory file for the different environments
3. Install jenkins on the control server. The steps to install jenkins are detailed in [Install Jenkins](https://www.jenkins.io/doc/book/installing/linux/):

```SHELL
sudo yum install wget -y
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo

sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum upgrade -y
# Add required dependencies for the jenkins package
sudo yum install java-11-openjdk -y 
sudo yum install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl daemon-reload
```

4. It is recommended you assign an elastic ip to the jenkins server. Ensure port 8080 is open on the jenkins server, Setup jenkins by accessing the jenkins ui using <public_ip:8080>, get your set password by running "sudo cat /var/lib/jenkins/secrets/initialAdminPassword", install recommended pluggins, create and account and start using jenkins.

5. Install & Open Blue Ocean Jenkins Plugin. Go to "Manage Jenkins"> Manage pluggin> click on "Available Pluggins > Search for "Blue Ocean" and install without restart.
6. Create a new pipeline by going back to the Jenkins Dashboard, click on the "Blue Ocean" icon and click on "Create a new pipeline". 

![](./img/open_blue_ocean.png)

7. Select GitHub
8. On the "Connect to Github section, click on "Create an access token" to login to GitHub & Generate an Access token.
9. Enter your github access token
10. Click on your github account and select the "ansible-config" repository to complete the setup.

At this point you may not have a Jenkinsfile in the Ansible repository, so Blue Ocean will attempt to give you some guidance to create one. But we do not need that. We will rather create one ourselves. So, click on "Administration" to exit the Blue Ocean console.

![](./img/exit_blueocean.png)

At the stage, we have created a new pipeline. It takes the name of your GitHub repository.

![](./img/createdPipeline.png)

11. The goal is to run the ansible playbooks using jenkins. To do this we need to create a "Jenkinsfile". Inside the Ansible-config directory, create a new directory called "deploy" and create a new file call "Jenkinsfile" inside this directory.

12. Add the code snippet below to start building the Jenkinsfile gradually. This pipeline currently has just one stage called Build and the only thing we are doing is using the shell script module to echo Building Stage

```SHELL
pipeline {
    agent any

  stages {
    stage('Build') {
      steps {
        script {
          sh 'echo "Building Stage"'
        }
      }
    }
    }
}
```
13. Commit and push these changes to github
14. Now go back into the Ansible pipeline ("ansible-config") in Jenkins, and select "configure"
15. Scroll down to "Build Configuration" section and specify the location of the Jenkinsfile at deploy/Jenkinsfile, the save this change. This will automatically scan the github repo to identify the branches and files in the repo.

![](./img/build_config.png)

16. Then go back to jenkins and in the pipeline again, click "Build now" to run the pipeline. This would fetch the code from github and run the simple pipeline we added to the jenkins file.

This will trigger a build and you will be able to see the effect of our basic Jenkinsfile configuration by going through the console output of the build.

![](./img/first_pipeline_job.png)

To really appreciate and feel the difference of Blue Ocean UI, we will be triggering the build again from Blue Ocean interface.

* Click on Blue Ocean, select the "ansible-config" job, click on the branch (this will be "main" at this point) and click on the play icon.

![](./img/blue_ocean_run.png)

Notice that this pipeline is a multibranch one. This means, if there were more than one branch in GitHub, Jenkins would have scanned the repository to discover them all and we would have been able to trigger a build for each branch.

To demonstrate this:

* Create a new git branch and name it feature/jenkinspipeline-stages
* Currently we only have the Build stage. Let us add another stage called Test. Paste the code snippet below and push the new changes to GitHub.

```SHELL
pipeline {
  agent any
  
  stages {
    stage('Build') {
      steps {
        script {
          sh 'echo "Building Stage"'
        }
      }
    }

    stage('Test') {
      steps {
        script {
          sh 'echo "Testing Stage"'
        }
      }
    }
    }
}
```
* Commit and push these changes to github.
* To make your new branch show up in Jenkins, we need to tell Jenkins to scan the repository by clicking on the "Administration" button and the the "Scan Repository Now" icon.

![](./img/scan_now.png)

* Refresh the page and both branches will start building automatically. You can go into Blue Ocean and see both branches there too.
* In Blue Ocean, you can now see how the Jenkinsfile has caused a new step in the pipeline launch build for the new branch.



```SHELL
touch dev && touch uat && touch pentest && touch ci && touch preprod && touch prod && touch sit
```

3. Create the below instances and update the inventory files for ci and dev with the ip addresses:

- ci

nginx (RHEL 8, t2-micro), 
sonarqube (ubuntu, t2-medium),
artifactory (RHEL 8, t2-micro)


- dev

nginx, 
todo webserver (RHEL 8, t2-micro),
tooling webserver (RHEL 8, t2-micro)
db (ubuntu, t2-micro)

4. update the dev and ci inventory files with the private ip of the above servers

ci

```SHELL
jenkins-server ansible_host=<Private-IP-Address>
nginx-server ansible_host=<Private-IP-Address>
sonarqube-server ansible_host=<Private-IP-Address>
artifactory-server ansible_host=<Private-IP-Address>

[jenkins]
jenkins-server ansible_user=ec2-user

[nginx]
nginx-server ansible_user=ec2-user

[sonarqube]
sonarqube-server ansible_user=ubuntu

[artifactory]
artifactory-server ansible_user=ec2-user
```

dev

```SHELL
nginx-server ansible_host=<Private-IP-Address>
db-server ansible_host=<Private-IP-Address>
todo-server ansible_host=<Private-IP-Address>
tooling-server ansible_host=<Private-IP-Address>


[nginx]
nginx-server ansible_user=ec2-user

[db]
db-server ansible_user=ubuntu

[todo]
todo-server ansible_user=ec2-user

[tooling]
tooling-server ansible_user=ec2-user

[db:vars]
ansible_python_interpreter=/usr/bin/python
```

**STEP 2: Ansible Role for CI Environment**

Three roles are required for the ci environment so we will be adding the below roles. TO do this:

* Create a new directory called "roles" inside the "ansible-config" directory.
* Initialize the below three roles using ansible-galaxy

    * Sonarqube, Artifactory, nginx
* 








