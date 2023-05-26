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
1. Initialize a new git repo called "ansible-config"
2. In the ansible-config directory create an inventory directory with inventory file for the different environments
```SHELL
touch dev && touch uat && touch pentest && touch ci && touch preprod && touch prod && touch sit
```

- Create the below instances and update the inventory files for ci and dev with the ip addresses:

- ci

nginx (RHEL 8, t2-micro), 
sonarqube (ubuntu, t2-medium),
artifactory (RHEL 8, t2-micro)
todo webserver (RHEL 8, t2-micro),
tooling webserver (RHEL 8, t2-micro)
db (ubuntu, t2-micro)

- dev

nginx, 
todo webserver,
tooling webserver,
db

