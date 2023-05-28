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

**STEP 1: Set up Inventory files**

1. Create an inventory file for all environments

```SHELL
cd inventory
touch dev && touch uat && touch pentest && touch ci && touch preprod && touch prod && touch sit
```

2. Create the below instances and update the inventory files for ci and dev with the ip addresses:

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
```

__STEP 2: Setup Jenkins Configurations__

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

![](./img/two_branches.png)

* In Blue Ocean, you can now see how the Jenkinsfile has caused a new step in the pipeline launch build for the new branch.

![](./img/build_from_feature_branch.png)


Next we will carry out the below steps:

1. Create a pull request to merge the latest code into the main branch
2. After merging the Pull Request, go back into your terminal and switch into the main branch.
3. Pull the latest change.
4. Create a new branch, add more stages into the Jenkinsfile to simulate below phases. 
    1. Add a stage to clean up workspace at the beginning of the pipeline.
   2. Echo "Package"
   3. Echo "Deploy"
   4. Clean up

```SHELL
pipeline {
  agent any
  
  stages {
    stage("Initial Cleanup") {
      steps {
        dir("${WORKSPACE}") {
          deleteDir()
        }
      }
    }

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
    stage('Package') {
      steps {
        script {
          sh 'echo "Packaging Stage"'
        }
      }
    }
    stage('Deploy') {
      steps {
        script {
          sh 'echo "Deploying Stage"'
        }
      }
    }
    stage('Clean Up') {
      steps {
        cleanWs(cleanWhenAborted: true, cleanWhenFailure: true, cleanWhenNotBuilt: true, cleanWhenUnstable: true, deleteDirs: true)
      }
    }
  }
}
```

5. Verify in Blue Ocean that all the stages are working, then merge your feature branch to the main branch

![](./img/additional_stages_in_branch.png)

6. Eventually, your main branch should have a successful pipeline like this in blue ocean

![](./img/additional_stages-in_main.png)


**STEP 3: Install Ansible and Setup Ansible Role for CI Environment**

Three roles are required for the ci environment so we will be adding the below roles. To do this:

* Install ansible on the jenkins server.

```Shell
sudo yum install ansible -y
```

* Install ansible dependencies to enable us run mysql and postgresql using ansible.

```SHELL
sudo yum install python3 python3-pip wget unzip git -y

# Upgrade pip package
sudo python3 -m pip install --upgrade setuptools
sudo python3 -m pip install --upgrade pip

# Dependencies to run sql commands
sudo python3 -m pip install PyMySQL
sudo python3 -m pip install mysql-connector-python
sudo python3 -m pip install psycopg2==2.7.5 --ignore-installed

# For mysql db
ansible-galaxy collection install community.mysql
```

* Create a new directory called "roles" inside the "ansible-config" directory.
* Initialize the below three roles using ansible-galaxy

    * Sonarqube, Artifactory, nginx and mysql

```SHELL
cd roles
ansible-galaxy init sonarqube
ansible-galaxy init artifactory
ansible-galaxy install geerlingguy.nginx -p . && mv geerlingguy.nginx/ nginxRole
ansible-galaxy install geerlingguy.mysql -p . && mv geerlingguy.mysql/ ./mysql
```

**Why do we need SonarQube?**

SonarQube is an open-source platform developed by SonarSource for continuous inspection of code quality, it is used to perform automatic reviews with static analysis of code to detect bugs, code smells, and security vulnerabilities. [Watch a short description here](https://youtu.be/vE39Fg8pvZg). There is a lot more hands on work ahead with SonarQube and Jenkins. So, the purpose of SonarQube will be clearer to you very soon.

Why do we need Artifactory?
Artifactory is a product by JFrog that serves as a binary repository manager. The binary repository is a natural extension to the source code repository, in that the outcome of your build process is stored. It can be used for certain other automation, but we will it strictly to manage our build artifacts.

[Watch a short description here](https://youtu.be/upJS4R6SbgM) Focus more on the first 10.08 mins

**STEP 4: Running Ansible Playbook from Jenkins**

Now that you have a broad overview of a typical Jenkins pipeline. Let us get the actual Ansible deployment to work by:

1. Installing Ansible plugin in Jenkins UI: Navigate to Dashboard>Manage Jenkins>Plugin Manager>Available plugins and search for "Ansible". Install without restart.

![](./img/install_ansible_pluggin.png)

Go back to "Dashboard>Manage Jenkins>Global Tools Configuration" and enter the below configuration under "Ansible"

![](./img/set_ansible_path.png)

2. Creating Jenkinsfile from scratch. (Delete all you currently have in there and start all over to get Ansible to run successfully). This should take care of the following considerations:

* This jenkinsfile should be parameterized to run against different environments without editting the file directly.
* It should include a stage to checkout the SCM to a specified branch.
* Jenkins should export the ANSIBLE_CONFIG environment variable. ansible.cfg file should be added to the "deploy" folder alongside the Jenkinsfile. This way, anyone can easily tell that everything in there are related to deployment.
* Dynamically set the environment variables (very importantly, the Role variable) each time the job runs.
* This pipeline should always delete the already existing files generated from previous runs. This is done by adding a clean up step at the begining of the script.
* Using the "Pipeline Syntax tool in jenkins, generate the syntax to create environment variables to set dynamically each time the pipeline is ran and run the ansible-playbook.
* A final step to clean up the workspace after build.

To run the ansible playbooks against the servers, jenkins would need to ssh into the server and it requires the private keys of the servers to do this. I used just one key for all servers so I need to add this key to jenkins credentials. To do this, navigate to "Manage Jenkins>Credentials>Global Credentials>Add Credentials" and configure it as shown below. In the "Key" section, copy your private key and add it to the box below. Jenkins would use this credentials to connect to the servers while running the ansible playbook.

![](./img/add_private_key.png)

**Ansible.cfg file**

```SHELL
[defaults]
timeout = 160
callback_whitelist = profile_tasks
log_path=~/ansible.log
host_key_checking = False
gathering = smart
ansible_python_interpreter=/usr/bin/python3
allow_world_readable_tmpfiles=true


[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=30m -o ControlPath=/tmp/ansible-ssh-%h-%p-%r -o ServerAliveInterval=60 -o ServerAliveCountMax=60 -o ForwardAgent=yes
```

**Jenkinsfile**

```SHELL
pipeline {
  agent any

  environment {
      ANSIBLE_CONFIG="${WORKSPACE}/deploy/ansible.cfg"
    }

  parameters {
      string(name: 'inventory', defaultValue: 'dev',  description: 'This is the inventory file for the environment to deploy configuration')
      string(name: 'gitBranch', defaultValue: 'feature/jenkinspipeline-stages', description: 'SCM Git branch to checkout')
    }

  stages{
      stage("Initial cleanup") {
          steps {
            dir("${WORKSPACE}") {
              deleteDir()
            }
          }
        }

      stage('Checkout SCM') {
         steps{
            git branch: "${gitBranch}", url: 'https://github.com/doutimi3/ansible-config.git'
         }
       }

      stage('Prepare Ansible For Execution') {
        steps {
          sh 'echo ${WORKSPACE}' 
          sh 'sed -i "3 a roles_path=${WORKSPACE}/roles" ${WORKSPACE}/deploy/ansible.cfg'  
        }
     }

      stage('Run Ansible playbook') {
        steps {
           ansiblePlaybook become: true, colorized: true, credentialsId: 'private-key', disableHostKeyChecking: true, installation: 'ansible', inventory: 'inventory/${inventory}', playbook: 'playbooks/site.yml'
         }
      }

      stage('Clean Workspace after build'){
        steps{
          cleanWs(cleanWhenAborted: true, cleanWhenFailure: true, cleanWhenNotBuilt: true, cleanWhenUnstable: true, deleteDirs: true)
        }
      }
   }

}
```

In the mysql role, navigate to "defaults>main.yml" and add the below block of code under the "Databases" section.

```YAML
# Databases.
mysql_databases:
  - name: tooling
    collation: utf8_general_ci
    encoding: utf8
    replicate: 1
```

Also add the below block of code under the "Users" section
```SHELL
# Users.
mysql_users:
  - name: webaccess
    host: 0.0.0.0
    password: secret
    priv: '*.*:ALL,GRANT'
```

In the nginxRole role, navigate to "defaults>main.yml" and clear all contents of this file.

Still in the nginxRole, navigate to "tasks>main.yml", clear all contents and add the below block of code:

```YAML
---
# tasks file for nginx
- name: install nginx on the webserver
  ansible.builtin.yum:
      name: nginx
      state: present


- name: ensure nginx is started and enabled
  ansible.builtin.service:
     name: nginx
     state: started 
     enabled: yes

- name: install PHP
  ansible.builtin.yum:
    name:
      - php 
      - php-mysqlnd
      - php-gd 
      - php-curl
    state: present
```
Still in the nginxRole, navigate to "tasks>main.yml", delete all other yml files aside from the "main.yml" file. 

Still in the nginxRole, navigate to "templates" directory, delete all configuration files and create a new file call "nginx.conf" with the below content. 

```SHELL
upstream backend {
      server  <private ip> weight=5; 
      server  <private ip> weight=5;
      
   }

   # This server accepts all traffic to port 80 and passes it to the upstream. 
   # Notice that the upstream name and the proxy_pass need to match.

   server {
      listen 80; 

      location / {
          proxy_pass http://backend;
      }
   }
```
* Create another folder called "static-assignments".
* Create a new file called nginx.yml inside the above folder and add the below lines of code:

```YAML
---
- hosts: nginx
  become: true
  roles:
     - nginxRole
```

* Add another file called "database.yml" inside the above folder with the below lines of code:
```YAML
---
- hosts: db
  roles:
    - mysql
```

* Create a new folder called "playbooks", inside this folder, create a file called "site.yml" and add the below block of code

```YAML
---
- hosts: db
- name: database assignment
  ansible.builtin.import_playbook: ../static-assignments/database.yml

- hosts: nginx
- name: nginx assignment
  ansible.builtin.import_playbook: ../static-assignments/nginx.yml
```

* Commit changes to the feature/pipeline-stages branch, confirm that this worked as expected on jenkins and create a pull request to merge it to main branch.

* On the Jenkins UI, navigate to "Dashboard>ansible-config" and click on "Scan repository Now". This will scan the repo to include the recent changes and trigger the build. The build might fail because we have not set the parameters for "inventory" and "gitBranch"

* Click on the "feature/jenkinspipeline-stages" branch and click on "Build with Parameters" to set the inventory and gitBranch parameters and click on "Build" to trigger the build.

![](./img/set_parameters.png)

This would run run all stages specified in the Jenkinsfile. See below the output of the stage that triggers the ansible-playbook to configure the nginx and database servers.


![](./img/pileline_stages1.png)

![](./img/Ansible_run.png)

Notice that we can now specify which environment we want to deploy the configuration to. Simply configure your "sit" inventory file, go to "Build with Parameters" and type "sit" and "Build" to run the pipeline against the sit environment.

As I mentioned earlier "The SIT (System Integration Testing) and UAT (User Acceptance Testing) environments are essentially the webservers holding the application so they don't need additional installation or configuration." So to avoid spinning up new servers for this I will be copying the contents of the dev inventory file into the sit inventory file.

```SHELL
[tooling]
<SIT-Tooling-Web-Server-Private-IP-Address>

[todo]
<SIT-Todo-Web-Server-Private-IP-Address>

[nginx]
<SIT-Nginx-Private-IP-Address>

[db]
<SIT-DB-Server-Private-IP-Address>

[db:vars]
ansible_user=ec2-user
```

To run this against the sit environment, we first need to setup the webserver ansible configurations. To do this:

* Under the "roles" directory, create a new role called "webserver"
```SHELL
ansible-galaxy init webserver
```
* Navigate to "Roles > webserver > tasks > main.yml" and add the below block of codes:

```YAML
---
# tasks file for webserver
- name: install apache
  become: true
  yum:
    name: "httpd"
    state: present

- name: install git
  become: true
  yum:
    name: "git"
    state: present

- name: clone a repo
  become: true
  git:
    repo: https://github.com/doutimi3/devops_tooling.git
    dest: /var/www/html
    force: yes

- name: copy html content to one level up
  become: true
  command: cp -r /var/www/html/html/ /var/www/

- name: Start service httpd, if not started
  become: true
  ansible.builtin.service:
    name: httpd
    state: started

- name: recursively remove /var/www/html/html/ directory
  become: true
  ansible.builtin.file:
    path: /var/www/html/html
    state: absent
```
* Navigate to "Roles > webserver > templates" directory, create a new file file called "apache-conf.j2" and add the below block of codes:

```SHELL
<VirtualHost *:80>
    ServerAdmin webmaster@{{ domain }}
    ServerName {{ domain }}
    ServerAlias www.{{ domain }}
    DocumentRoot /var/www/html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
* Navigate to "static-assignment" directory and create a new file called "webserver.yml" with the below content:

```YAML
---
- hosts: tooling
  roles:
    - webserver
```

* Navigate to "playbooks" directory and add the below lines of code to the "site.yml" file:

```YAML
- hosts: tooling
- name: deploy tooling website
  ansible.builtin.import_playbook: ../static-assignments/webserver.yml
```

* Commit and push code to github, on the jenkins UI scan the repo again and build job with parameter. This time set the environment to "sit"

![](./img/sit_run.png)

![](./img/tooling_set.png)

**STEP 4: CI/CD Pipeline for Todo Application**

Here we will introduce another PHP application to add to the list of software products we are managing in our infrastructure. The good thing with this particular application is that it has unit tests, and it is an ideal application to show an end-to-end CI/CD pipeline for a particular application.


* **Update Artifactory Role:** Our goal here is to deploy the application onto servers directly from Artifactory rather than from git. If you have not updated Ansible with an Artifactory role, simply use this guide to create an Ansible role for Artifactory (ignore the Nginx part). [Configure Artifactory on Ubuntu 20.04](https://www.howtoforge.com/tutorial/ubuntu-jfrog/). 

* Launch a t2.medium RHEL 8 instance and update the ci inventory file with the server private ip.
* Navigate to the "ansible-config directory > Roles" and create a new role called "artifactory" using ansible-galaxy if this have not been created already.
* Delete the following directories in the "artifactory directory": files, meta, tests, vars.
* Update the "Defaults > main.yml" with the below block of code:
```YAML
# defaults file for artifactory

# The version of artifactory to install
artifactory_version: 7.24.3

# Set this to true when SSL is enabled (to use artifactory_nginx_ssl role), default to false (implies artifactory uses artifactory_nginx role )
artifactory_nginx_ssl_enabled: false

# Set this to false when ngnix is disabled, defaults to true (implies artifactory uses artifactory_nginx role )
artifactory_nginx_enabled: false

# Provide single node license
# artifactory_single_license:

# Provide individual (HA) licenses file separated by new line and 2-space indentation and set artifactory_ha_enabled: true.
# Example:
# artifactory_licenses: |-
#   <license_1>

#   <license_2>

#   <license_3>

# To enable HA, set to true
artifactory_ha_enabled: false

# By default, all nodes are primary (CNHA) - https://www.jfrog.com/confluence/display/JFROG/High+Availability#HighAvailability-Cloud-NativeHighAvailability
artifactory_taskAffinity: any

# The location where Artifactory should install
jfrog_home_directory: /opt/jfrog

# Pick the Artifactory flavour to install, can be also cpp-ce/jcr/pro
artifactory_flavour: pro

artifactory_extra_java_opts: -server -Xms512m -Xmx2g -Xss256k -XX:+UseG1GC
artifactory_system_yaml_template: system.yaml.j2
artifactory_tar_file_name: jfrog-artifactory-pro-{{ artifactory_version }}-linux.tar.gz
artifactory_home: "{{ jfrog_home_directory }}/artifactory"
artifactory_tar: https://releases.jfrog.io/artifactory/artifactory-pro/org/artifactory/pro/jfrog-artifactory-pro/{{ artifactory_version }}/{{ artifactory_tar_file_name }}
artifactory_untar_home: "{{ jfrog_home_directory }}/artifactory-{{ artifactory_flavour }}-{{ artifactory_version }}"

# Timeout in seconds for URL request
artifactory_download_timeout: 10

postgres_driver_version: 42.2.23
postgres_driver_download_url: https://repo1.maven.org/maven2/org/postgresql/postgresql/{{ postgres_driver_version }}/postgresql-{{ postgres_driver_version }}.jar

artifactory_user: artifactory
artifactory_group: artifactory

artifactory_daemon: artifactory

artifactory_uid: 1030
artifactory_gid: 1030

# if this is an upgrade
artifactory_upgrade_only: false

#default username and password
artifactory_admin_username: admin
artifactory_admin_password: password

artifactory_service_file: /lib/systemd/system/artifactory.service

# Provide binarystore XML content below with 2-space indentation
artifactory_binarystore: |-
  <?xml version="1.0" encoding="UTF-8"?>
  <config version="2">
      <chain template="cluster-file-system"/>
  </config>

# Provide systemyaml content below with 2-space indentation
artifactory_systemyaml: |-
  configVersion: 1
  shared:
    security:
      joinKey: "{{ join_key }}"
    extraJavaOpts: "{{ artifactory_extra_java_opts }}"
    node:
      id: {{ ansible_hostname }}
      ip: {{ ansible_host }}
      taskAffinity: {{ artifactory_taskAffinity }}
      haEnabled: {{ artifactory_ha_enabled }}
    database:
      type: "{{ artifactory_db_type }}"
      driver: "{{ artifactory_db_driver }}"
      url: "{{ artifactory_db_url }}"
      username: "{{ artifactory_db_user }}"
      password: "{{ artifactory_db_password }}"
  router:
    entrypoints:
      internalPort: 8046

# Note: artifactory_systemyaml_override is by default false,  if you want to change default artifactory_systemyaml
artifactory_systemyaml_override: false
```
* Under "templates", add a new file called "bash-profile.j2" and update it with the below text to export environment variables:

```SHELL
export JAVA_HOME=$(dirname $(dirname $(readlink $(readlink $(which javac)))))
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/jre/lib:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
```

* Update "tasks > main.yml" with the below block of code:
```YAML
---
# tasks file for artifactory

- name: install java 11
  ansible.builtin.yum:
    name: java-11-openjdk-devel
    state: present

- name: install java 11
  ansible.builtin.yum:
    name: 
      - wget
      - unzip
    state: present

- name: Configuring java path
  ansible.builtin.template:
    src: templates/bash-profile.j2
    dest: .bash_profile
  

- name: reload the /etc/profile
  shell: source ~/.bash_profile


- name: add the repository key to repos list
  ansible.builtin.get_url:
    url:  https://releases.jfrog.io/artifactory/artifactory-rpms/artifactory-rpms.repo 
    dest: /home/ec2-user/jfrog-artifactory-rpms.repo
    mode: '0755'
 
- name: Copy the downloaded file to the etc repo
  ansible.builtin.copy:
    src: /home/ec2-user/jfrog-artifactory-rpms.repo
    dest: /etc/yum.repos.d/jfrog-artifactory-rpms.repo
    remote_src: yes
    follow: yes

- name: update cache
  ansible.builtin.yum:
    update_cache: yes

- name: install artifactory
  ansible.builtin.yum:
    name: jfrog-artifactory-oss
    state: present

- name: start and enable artifactory
  ansible.builtin.service:
    name: artifactory
    state: started
    enabled: yes
```
* Update "handlers > main.yml" with the below block of code:
```YAML
---
# handlers file for distribution
- name: restart artifactory
  become: yes
  systemd:
    name: "{{ artifactory_daemon }}"
    state: restarted

```
* Navigate to "ansible-config > static-assignments" directory and create a new file called "artifactory.yml", update it with the below block of code:

```YAML
---
- hosts: artifactory
  become: true
  roles:
    - artifactory
```
* Navigate to "ansible-config > playbooks" directory and append the below lines to "site.yml":

```YAML
- hosts: artifactory
- name: artifactory assignment
  ansible.builtin.import_playbook: ../static-assignments/artifactory.yml
```
* Commit and push changes to github.

* On the jenkins UI, scan the repository again, change the inventory parameter to "ci" and run the build against the ci environment.

![](./img/installed_artifactory.png)

* Grab the public ip of the articatory server, ensure port 8081 is open on the artifactory server and accesss jfrog artifactory using the url in this format: http://<public_ip>/:8081

![](./img/jfrog_login.png)

Username: admin
default password: password

* Login, click on "Get started" and set a new password of your choice, skip the next steps and click on "finish".

* Next, we will create a local repository to store the artifacts for the Todo application. 

  * Click on "Create a repository" > "Add repository" > Select "Local" > Click on "Generic" and enter the name of the application as the "Repository key" and leave all other settings as default. Save and close:

![](./img/setup_localRepo_artifactory.png)


* **Prepare Jenkins:**
1. Fork the repository below into your GitHub account
```SHELL
https://github.com/darey-devops/php-todo.git
```
2. Clone the repository to your local machine
```SHELL
git clone https://github.com/doutimi3/php-todo.git
```

3. On the Jenkins server, install PHP, its dependencies and Composer tool (Feel free to do this manually at first, then update your Ansible accordingly later)

```SHELL
# Install php and dependencies
# Add EPEL package repo and install Remi repo
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo yum module reset php -y
sudo yum module enable php:remi-7.4 -y
sudo yum install -y php php-common php-mbstring php-opcache php-intl php-xml php-gd php-curl php-mysqlnd php-fpm php-json
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
```

Install Composer and check the version installed

```SHELL
# Install composer
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/bin/composer
composer --version
```
4. Install Jenkins plugins
* **Plot plugin:** This will be used to display tests reports and code coverage information.
To install, On the Jenkins UI, navigate to "Dashboard > Manage Jenkins > Manage Plugins > Available Plugins" and search for "plot", install without restart.

* **Artifactory plugin:** This will be used to easily upload code artifacts into an Artifactory server. To install, on the Jenkins UI, navigate to "Dashboard > Manage Jenkins > Manage Plugins > Available Plugins" and search for "Artifactory", install without restart.

5. In Jenkins UI configure Artifactory

* On the jenkins UI navigate to "Dashboard > Manage Jenkins > Configure System", scroll down to "JFrog" section, click on "Add JFrog Instance" and Configure the server ID, URL, Credentials,and run Test Connection.

![](./img/Jenkins_jfrog_config.png)

**Integrate Artifactory repository with Jenkins**
* Open the php-todo directory on a new terminal and create a Jenkinsfile inside the php-todo directory. Update it with the below clode of code:

```SHELL
pipeline {
    agent any

  stages {

     stage("Initial cleanup") {
          steps {
            dir("${WORKSPACE}") {
              deleteDir()
            }
          }
        }

    stage('Checkout SCM') {
      steps {
            git branch: 'main', url: 'https://github.com/doutimi3/php-todo.git'
      }
    }

    stage('Prepare Dependencies') {
      steps {
             sh 'mv .env.sample .env'
             sh 'composer install'
             sh 'php artisan migrate'
             sh 'php artisan db:seed'
             sh 'php artisan key:generate'
      }
    }

    stage('Execute Unit Tests') {
      steps {
        sh './vendor/bin/phpunit'
      }
    }

    stage('Code Analysis') {
      steps {
        sh 'phploc app/ --log-csv build/logs/phploc.csv'
      }
    }

    stage('Plot Code Coverage Report') {
      steps {

            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Lines of Code (LOC),Comment Lines of Code (CLOC),Non-Comment Lines of Code (NCLOC),Logical Lines of Code (LLOC)                          ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'A - Lines of code', yaxis: 'Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Directories,Files,Namespaces', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'B - Structures Containers', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Average Class Length (LLOC),Average Method Length (LLOC),Average Function Length (LLOC)', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'C - Average Length', yaxis: 'Average Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Cyclomatic Complexity / Lines of Code,Cyclomatic Complexity / Number of Methods ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'D - Relative Cyclomatic Complexity', yaxis: 'Cyclomatic Complexity by Structure'      
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Classes,Abstract Classes,Concrete Classes', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'E - Types of Classes', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Methods,Non-Static Methods,Static Methods,Public Methods,Non-Public Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'F - Types of Methods', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Constants,Global Constants,Class Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'G - Types of Constants', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Test Classes,Test Methods', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'I - Testing', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Logical Lines of Code (LLOC),Classes Length (LLOC),Functions Length (LLOC),LLOC outside functions or classes ', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'AB - Code Structure by Logical Lines of Code', yaxis: 'Logical Lines of Code'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Functions,Named Functions,Anonymous Functions', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'H - Types of Functions', yaxis: 'Count'
            plot csvFileName: 'plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv', csvSeries: [[displayTableFlag: false, exclusionValues: 'Interfaces,Traits,Classes,Methods,Functions,Constants', file: 'build/logs/phploc.csv', inclusionFlag: 'INCLUDE_BY_STRING', url: '']], group: 'phploc', numBuilds: '100', style: 'line', title: 'BB - Structure Objects', yaxis: 'Count'

      }
    } 

    stage ('Package Artifact') {
    steps {
            sh 'zip -qr php-todo.zip ${WORKSPACE}/*'
     }
    } 

    stage ('Upload Artifact to Artifactory') {
          steps {
            script { 
                 def server = Artifactory.server 'artifactory'                 
                 def uploadSpec = '''{
                    "files": [
                      {
                       "pattern": "php-todo.zip",
                       "target": "TodoApp/php-todo",
                       "props": "type=zip;status=ready"
                       }
                    ]
                 }'''

                 server.upload spec: uploadSpec
               }
            } 

        }

    stage ('Deploy to Dev Environment') {
      steps {
        build job: 'ansible-config/main', parameters: [[$class: 'StringParameterValue', name: 'env', value: 'dev']], propagate: false, wait: true
            }
        }
}
}

```
* php-todo app needs to connect to the database, so we will create a new database and user in the database for this application. This will be done using ansible. To do this, navigate back to the "ansible-config" directory, under Roles, navigate to the "mysql" role and in "defaults > main.yml", scroll down to the "databases" and "Users" section add the below line of codes:

The host of the user must be the private ip address of the jenkins server. This is because the jenkins server would act as a client to connect to the database server.

```SHELL
# Database
mysql_databases:
  - name: homestead
    collation: utf8_general_ci
    encoding: utf8
    replicate: 1

# User
mysql_users:
  - name: homestead
    host: 172.31.39.95
    password: Password@123
    priv: '*.*:ALL,GRANT'
```

* Commit these changes to the ansible-config directory and run the build on the jenkins UI.

* Verify that this ran successfully
![](./img/verify_db_user_creation_ansible.png)

![](./img/verify_homestead_user_db_access.png)

* Navigate back to the "php-todo" directory, commit and push changes to githbub.

* Go the the Jenkins UI and create a new pipeline, select the "php-todo" as the github repo for this pipeline. Because we already have a Jenkinsfile in this repo, this will attempt to run the build for it will fail because we have not installed some dependencies on the Jenkins server.

Notice that in the "Prepare Dependencies section" of the Jenkinsfile

The required file by PHP is .env so we are renaming .env.sample to .env
Composer is used by PHP to install all the dependent libraries used by the application
php artisan uses the .env file to setup the required database objects – (After successful run of this step, login to the database, run show tables and you will see the tables being created for you)

  * In the php-todo directory, update the ".env.sample" file with your database connection details:

DB_Host: Private IP of database server.


  ```SHELL
DB_HOST=172.31.47.92
DB_DATABASE=homestead
DB_USERNAME=homestead
DB_PASSWORD=Password@123
DB_CONNECTION=mysql
DB_PORT=3306
```

* Ensure the "binding Address" in the mysql config file (/etc/mysql/mysql.conf.d/mysqld.cnf) have been changed to 0.0.0.0 to allow connection from any ip address and mysql server have been restarted if this config file was modified.

* Install phpunit and phploc: This will be used by the code analysis stage of our pipeline:

```SHELL
sudo dnf --enablerepo=remi install php-phpunit-phploc -y
sudo wget -O phpunit https://phar.phpunit.de/phpunit-7.phar
chmod +x phpunit
sudo yum install php-xdebug -y
sudo yum install zip -y
```

* Enable xdebug: 
  * Use the command mentioned earlier (php --ini | grep "php.ini") to find the location of the php.ini file.
  * Once you have located the php.ini file, open it using a text editor.
  * Look for the xdebug section in the php.ini file. If it doesn't exist, you may need to add it manually. Within the xdebug section, find or add the xdebug.mode directive and set it to 'coverage'. It should look like this:

    ```SHELL
    xdebug.mode = coverage
    ```
  * restart PHP-FPM (sudo systemctl restart php-fpm)

* Push changes to github and build the php-todo pipeline


* Go to aws console and create a RHEL 8 "todo-webserver" of t2.micro instance type and update the dev inventory file with the private ip of the is server. The dev inventory file should look like this now:

```SHELL
nginx-server ansible_host=<private_ip>
db-server ansible_host=<private_ip>
todo-server ansible_host=<private_ip>
tooling-server ansible_host=<private_ip>


[nginx]
nginx-server ansible_user=ec2-user

[db]
db-server ansible_user=ubuntu

[todo]
todo-server ansible_user=ec2-user

[tooling]
tooling-server ansible_user=ec2-user
```

* Commit and Push changes to github for the ansible-config.
* Create a pull request in the ansible-config repo to merge the feature/jenkinspipeline-stage branch to the main branch. 
* Commit and push changes to php-todo repository and build the php-todo pipeline.

The build job used in this step tells Jenkins to start another job. In this case it is the ansible-project job, and we are targeting the main branch. Hence, we have ansible-project/main. Since the Ansible project requires parameters to be passed in, we have included this by specifying the parameters section. The name of the parameter is env and its value is dev. Meaning, deploy to the Development environment.

![](./img/successful_build.png)

**Explanation of what Happened in each Stage**

**Initial Cleanup:** This cleaned up the workspace to delete all files generated from previous runs

**Checkout SCM:** This check out the main branch of the "php-todo" repository that contains all source code for the TodoApp.

![](./img/SCM_stage.png)

**Prepare Dependencies:** In this stage, 

  * The .env.sample file is renamed to .env because php requires .env. 
  * Composer is installed. Composer is used by PHP to install all the dependent libraries used by the application.
  * php artisan uses the .env file to setup the required database objects

![](./img/Prep_dependencies_stage.png)

**Execute Unit Tests:** This step is using the "sh" (shell) command to execute a shell script. The script being executed is ./vendor/bin/phpunit. This command is commonly used in PHP projects to run unit tests using the PHPUnit testing framework.

![](./img/exec_unit_tests_stage.png)

**Code Analysis:** The step is using the "sh" command to execute a shell script. The script being executed is phploc app/ --log-csv build/logs/phploc.csv.

For PHP the most commonly tool used for code quality analysis is phploc. In this case, it is being used to analyze the code located in the app/ directory. The --log-csv option specifies that the analysis results should be saved in CSV format, and the build/logs/phploc.csv argument provides the path where the CSV file will be saved.

This stage is responsible for performing code analysis on the PHP project using phploc and generating a CSV file with the analysis results. Click on this link to learn more about the [phploc](https://matthiasnoback.nl/2019/09/using-phploc-for-quick-code-quality-estimation-part-1/).

![](./img/code_analysis_stage.png)

**Plot Code Coverage Report:** This stage uses the plot pluggin for jenkins to plot the results of the code analysis. Within this stage, there are multiple plot steps, each generating a different chart or plot based on the data provided in a CSV file (plot-396c4a6b-b573-41e5-85d8-73613b2ffffb.csv). These plots represent different aspects of code analysis and metrics. Here is a breakdown of the plots generated:

  * Plot A: Displays the "Lines of code" metric.
  * Plot B: Displays the count of various structure containers like directories, files, and namespaces.
  * Plot C: Shows the average length of classes, methods, and functions.
  * Plot D: Represents the relative cyclomatic complexity.
  * Plot E: Displays the count of different types of classes.
  * Plot F: Shows the count of different types of methods.
  * Plot G: Displays the count of different types of constants
  * Plot I: Represents the count of test classes and methods.
  * Plot AB: Represents the code structure based on logical lines of code.
  * Plot H: Displays the count of different types of functions.
  * Plot BB: Represents the count of different structure objects like interfaces, traits, classes, methods, functions, and constants.

Each plot step specifies the CSV file containing the data (build/logs/phploc.csv) and provides additional configuration options for customizing the plot, such as the plot title, y-axis label, plot style, inclusion and exclusion filters, and the number of builds to consider.

These plots help visualize and analyze different code metrics and provide insights into the structure, complexity, and test coverage of the PHP project.

You should now see a Plot menu item on the left menu. Click on it to see the charts. (The analytics may not mean much to you as it is meant to be read by developers. So, you need not worry much about it – this is just to give you an idea of the real-world implementation).

![](./img/code_analysis_plots.png)

**Package Artifacts:** This stage is responsible for packaging the project's workspace into a ZIP archive named php-todo.zip. The resulting ZIP archive can be used as an artifact, which can be archived, published, or deployed to a different location for further use or distribution.

![](./img/package_artifact_stage.png)

**Upload Artifact to Artifactory:** This stage allowed us to upload the previously created "php-todo.zip" artifact to an Artifactory server, specifically to the "TodoApp/php-todo" path, and assigns specific properties to the uploaded file. This step ensures that the artifact is stored in the designated location within the Artifactory repository for further use or distribution.

  * The Artifactory.server method is used to define a server configuration named 'artifactory'. This configuration should be preconfigured in the Jenkins instance or loaded from a Jenkins configuration file.
  * An upload specification (uploadSpec) is defined as a multi-line string using triple quotes ('''). This specification defines the files to be uploaded and their destination in the Artifactory repository. In this case, it specifies a single file named "php-todo.zip" with a target path of "TodoApp/php-todo" in the Artifactory repository. Additionally, it sets properties for the uploaded file, including "type=zip" and "status=ready".
  * The server.upload method is called, passing the upload specification (uploadSpec) as a parameter. This method initiates the upload of the specified file to the Artifactory server using the configured server details and the provided upload specification.

![](./img/upload_artifact_stage.png)

We can confirm in the artifactory server to see the uploaded artifact.

![](./img/todo_artifact.png)

**Deploy to Dev Environment:** This stage triggers the execution of the "ansible-config/main" job with the specified 'dev' environment parameter. It waits for the job to complete before proceeding with the pipeline execution. This stage is responsible for deploying the application to the development environment using Ansible configurations in the "ansible-config/main" job.

![](./img/deploy_to_dev_stage.png)

Notice that at this stage, we have not included an ansible playbook to set up the todo webserver, download the artifact from the artifactory repo and deploy the app to the webserver. The goal is to deploy the Todo App so we need to create a new ansible-playbook to configure the webserver and deploy the Todo App and modify the site.yml file to call this playbook and execute the tasks.

* Navigate to the "ansible-config" directory, under the "static-assignments" directory, create a new yml file called "deployment.yml" and add the below block of codes to this file:

```YAML
---
- name: Deploying the PHP Applicaion to Dev Enviroment
  become: true
  hosts: todo

  tasks:
    - name: install remi and rhel repo
      ansible.builtin.yum:
        name: 
          - https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
          - dnf-utils
          - http://rpms.remirepo.net/enterprise/remi-release-8.rpm
        disable_gpg_check: yes

    
    - name: install httpd on the webserver
      ansible.builtin.yum:
        name: httpd
        state: present

    - name: ensure httpd is started and enabled
      ansible.builtin.service:
        name: httpd
        state: started 
        enabled: yes
      
    - name: install PHP
      ansible.builtin.yum:
        name:
          - php 
          - php-mysqlnd
          - php-gd 
          - php-curl
          - unzip
          - php-common
          - php-mbstring
          - php-opcache
          - php-intl
          - php-xml
          - php-fpm
          - php-json
        enablerepo: php:remi-7.4
        state: present
    
    - name: ensure php-fpm is started and enabled
      ansible.builtin.service:
        name: php-fpm
        state: started 
        enabled: yes

    - name: Download the artifact
      get_url:
        url: http://35.179.48.145:8082/artifactory/TodoApp/php-todo
        dest: /home/ec2-user/
        url_username: admin
        url_password: APBKm4qXoDaoE4pTfHAD6iPnJMh  


    - name: unzip the artifacts
      ansible.builtin.unarchive:
       src: /home/ec2-user/php-todo
       dest: /home/ec2-user/
       remote_src: yes

    - name: deploy the code
      ansible.builtin.copy:
        src: /home/ec2-user/var/lib/jenkins/workspace/php-todo_main/
        dest: /var/www/html/
        force: yes
        remote_src: yes

    - name: remove nginx default page
      ansible.builtin.file:
        path: /etc/httpd/conf.d/welcome.conf
        state: absent

    - name: restart httpd
      ansible.builtin.service:
        name: httpd
        state: restarted
```
**NB:** In the "Download Artifact" section, the url should be replaced with the url cfor your artifact, copied from the TodoApp repository that was created in artifactory to store the artifact.

![](./img/todo_artifact.png)

* To get the url_password (which is essentially the encrypted version of your artifactory password), click on "Set Me Up" in the top right hand corner of the page shown above, enter your password and click on "Generate Token", the copy the encrypted password and replace the url_password with it.

![](./img/encrypt_password.png)

* Navigate to the "playbooks" directory and add the below lines to the "site.yml" file:

```YAML
- hosts: todo
- name: Deploy the todo application
  ansible.builtin.import_playbook: ../static-assignments/deployment.yml
```

* Commit changes to the "feature/jenkinspipeline-stages" branch, push code to github and create a pull request to the main branch.

* Navigate to Jenkins and rerun the "php-todo" pipeline.

![](./img/deploy_phpApp_ansible.png)

So far, we have successfully deployed the Todo app to the dev environment. But how are we certain that the code deployed has the quality that meets corporate and customer requirements? Even though we have implemented Unit Tests and Code Coverage Analysis with phpunit and phploc, we still need to implement Quality Gate to ensure that ONLY code with the required code coverage, and other quality standards make it through to the environments.

To achieve this, we need to configure SonarQube – An open-source platform developed by SonarSource for continuous inspection of code quality to perform automatic reviews with static analysis of code to detect bugs, code smells, and security vulnerabilities.

**INSTALL SONARQUBE**

Before we start getting hands on with SonarQube configuration, it is incredibly important to understand a few concepts:

* **Software Quality:** The degree to which a software component, system or process meets specified requirements based on user needs and expectations.
* **Software Quality Gates:** Quality gates are basically acceptance criteria which are usually presented as a set of predefined quality criteria that a software development project must meet in order to proceed from one stage of its lifecycle to the next one.

SonarQube is a tool that can be used to create quality gates for software projects, and the ultimate goal is to be able to ship only quality software code.

Despite that DevOps CI/CD pipeline helps with fast software delivery, it is of the same importance to ensure the quality of such delivery. Hence, we will need SonarQube to set up Quality gates. In this project we will use predefined Quality Gates, also known as [The Sonar Way](https://docs.sonarqube.org/latest/instance-administration/quality-profiles/). Software testers and developers would normally work with project leads and architects to create custom quality gates.

**Install SonarQube on Ubuntu 20.04 With PostgreSQL as Backend Database**

* Create a new role called "sonarqube" in "ansible-config > Roles" directory.

```SHELL
cd ~/ansible-config/roles/
ansible-galaxy init sonarqube
cd sonarqube/
rm -rf test && rm -rf vars && rm -rf defaults && rm -rf files
```
* Create two additional files under "~/ansible-config/roles/sonarqube/tasks" called "postgresql.yml and sonarqube-setup.yml"

* Still in "~/ansible-config/roles/sonarqube/tasks", add the below block of codes to "main.yml" to install sonarqube and dependencies.

```YAML
---
# tasks file for sonarqube
- name: set max_map
  become: true
  command: sysctl -w vm.max_map_count=262144

- name: set file max
  become: true
  command: sysctl -w fs.file-max=65536

- name: Add max limits
  ansible.builtin.lineinfile:
    path: /etc/security/limits.conf
    line: 
      - sonarqube   -   nofile   65536
      - sonarqube   -   nproc    4096
    create: yes

- name: Add ppa  repository 
  ansible.builtin.apt_repository:
    repo: ppa:linuxuprising/java

- name: install python3
  ansible.builtin.apt:
    name: 
      - python3
      - python3-pip
      - python3-dev
      - libpq-dev
    state: present

- name: Install setuptools
  pip:
    name: setuptools
    extra_args: --upgrade

- name: upgrade pip command
  pip:
    name: pip
    extra_args: --upgrade

- name: install pip dependencies
  pip:
    name: psycopg2
    executable: pip3

- name: install java 11
  ansible.builtin.apt:
    name: 
      - openjdk-11-jdk
      - openjdk-11-jre
    state: present
    allow_unauthenticated: yes

- import_tasks: postgresql.yml

- name: Ensure group sonar exists
  ansible.builtin.group:
    name: sonar
    state: present

- import_tasks: sonarqube-setup.yml

- name: start and enable sonarqube
  ansible.builtin.service:
    name: sonar
    state: started
    enabled: yes
```

* Still in "~/ansible-config/roles/sonarqube/tasks", add the below block of codes to "postgresql.yml" to set up postgresql.

```YAML
- name: add PostgreSQL repo to the repo list
  ansible.builtin.lineinfile:
    path: /etc/apt/sources.list.d/pgdg.list
    line: deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main
    create: yes

- name: Download PostgreSQL software key
  ansible.builtin.apt_key:
    url: https://www.postgresql.org/media/keys/ACCC4CF8.asc
    state: present

- name: install postgresql
  ansible.builtin.apt:
    pkg:
    - postgresql
    - postgresql-contrib
    state: present

- name: start and enable postgresql
  ansible.builtin.service:
    name: postgresql
    state: started
    enabled: yes
     
- name: create password for user postgres
  ansible.builtin.user:
    name: postgres
    group: postgres
    password: postgres

- name: Create a new database sonarqubedb
  become_user: postgres
  community.postgresql.postgresql_db:
    name: sonarqube
    encoding: UTF-8

- name: Create user sonarqube
  become_user: postgres
  community.postgresql.postgresql_user:
    db: sonarqube
    name: sonar
    password: sonar
    priv: "ALL"
```
* Still in "~/ansible-config/roles/sonarqube/tasks", add the below block of codes to "sonarqube-setup.yml" to set up sonarqube.

```YAML
- name: install unzip and wget
  ansible.builtin.apt:
    name:
      - unzip
      - wget

- name: download the zip file
  ansible.builtin.get_url:
    url: https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-7.9.3.zip
    dest: /opt/
    mode: 0755

- name: Create /opt/sonarqube/
  ansible.builtin.file:
    path: /opt/sonarqube/
    state: directory
    mode: '0755'

- name: Extract the zip file
  ansible.builtin.unarchive:
    src: /opt/sonarqube-7.9.3.zip
    dest: /opt/
    remote_src: yes

- name: Copy /opt/sonarqube/
  ansible.builtin.copy:
    src: /opt/sonarqube-7.9.3/
    dest: /opt/sonarqube/
    remote_src: yes
    follow: yes

- name: remove the zip file
  ansible.builtin.file:
    path: /opt/sonarqube-7.9.3.zip
    state: absent

- name: add user to run sonar
  ansible.builtin.shell: useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar 

- name: change ownership
  ansible.builtin.shell: chown sonar:sonar /opt/sonarqube -R
    
- name: Configuring the SonarQube Server
  ansible.builtin.template:
    src: templates/sonar.properties.j2
    dest: /opt/sonarqube/conf/sonar.properties
    force: yes

- name: Update user in sonar.sh filee
  ansible.builtin.lineinfile:
    path: /opt/sonarqube/bin/linux-x86-64/sonar.sh
    regexp: '^#RUN_AS_USER='
    state: present
    line: RUN_AS_USER=sonar
    

- name: create a service file
  ansible.builtin.template:
    src: templates/sonarqube.service.j2
    dest:  /etc/systemd/system/sonar.service
    force: yes
```

Navigate to "~/ansible-config/roles/sonarqube/templates" and create two configuration files named "sonar.properties.j2 and sonarqube.service.j2". Update these files with the content of these files [sonar.properties.j2](./roles/sonarqube/templates/sonar.properties.j2) and [sonarqube.service.j2](./roles/sonarqube/templates/sonarqube.service.j2).

* Navigate to "~/ansible-config/static-assignments" and create a new file called "sonarqube.yml" and add the below contents to this file:

```YAML
---
- hosts: sonarqube
  become: true
  roles:
     - sonarqube
```

* Navigate to "~/ansible-config/playbooks/" and add the below lines to "site.yml"
```YAML
- hosts: sonarqube
- name: sonar assignment
  ansible.builtin.import_playbook: ../static-assignments/sonarqube.yml
```

* Commit code to github and run the build against the ci environment to setup the sonarqube server.

**Access SonarQube**

* Ensure port 9000 is open and confirm access to sonarqube by entering the public ip of the sonarqube server on your browser in this format http://13.41.70.79:9000/sonar/.

![](./img/confirm_sonar_access.png)

* Login to SonarQube with default administrator username and password – admin

![](./img/login_sonar.png)

![](./img/sonar_accessed.png)

**CONFIGURE SONARQUBE AND JENKINS FOR QUALITY GATE**

* In Jenkins, install "SonarQube Scanner" plugin

* Navigate to configure system in Jenkins. Add SonarQube server as shown below:

Manage Jenkins > Configure System

![](./img/add_sonarscanner.png)

* Generate authentication token in SonarQube

 User > My Account > Security > Generate Tokens

![](./img/Sonarqube-Token.png)

* Configure Quality Gate Jenkins Webhook in SonarQube – The URL should point to your Jenkins server http://{JENKINS_HOST}/sonarqube-webhook/

Administration > Configuration > Webhooks > Create
![](./img/Sonar-Jenkins-Webhook.png)

* Setup SonarQube scanner from Jenkins – Global Tool Configuration

Manage Jenkins > Global Tool Configuration

![](./img/Jenkins-SonarScanner.png)

* Update Jenkins Pipeline to include SonarQube scanning and Quality Gate by adding the below code to the Jenkinsfile. This should be placed before the artifact packaging stage.

```SHELL
    stage('SonarQube Quality Gate') {
        environment {
            scannerHome = tool 'SonarQubeScanner'
        }
        steps {
            withSonarQubeEnv('sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner"
            }

        }
    }
```

* Commit and push changes to github then run the "php-todo" pipeline. This would fail because we have not updated "sonar-properties". However, we still need to run the above step because it will install the scanner tool on the Linux server.

* Configure sonar-scanner.properties – From the step above, Jenkins will install the scanner tool on the Linux server. You will need to go into the tools directory on the server to configure the properties file in which SonarQube will require to function during pipeline execution.






































