###### DHS-HSSCCG-17-I-00018
<h1 align="center">
  <br>
  <a href="http://www.inadev.com/"><img src="http://inadev.com/sites/all/themes/inadev_2016/img/Inadev_Logo.png" alt="INADEV" width="200"></a>
  <br>
  <br>
  <br>
  <br>
Instructions to evaluate INADEV’s CI/ CD pipeline
  <br>
  <br>
</h1>
  <br>
  <br>
  
## Workflow Description

<br>

![ CI/CD Workflow Diagram](http://54.235.108.125/RFI.png)

<br>

## Evaluate Workflow

In order to evaluate our automation code, it is required that you make the following changes to the various components:

### A. CloudFormation Template

We have designed a CloudFormation template to launch the whole "Hello World" Application infrastructure.   
When the CloudFormation template is imported into AWS CloudFormation, you will be asked to provide the following details:

<table>
  <tr>
    <th>Parameters</th>
    <th>Details</th>
  </tr>
  <tr>
    <td>envPrefix</td>
    <td>This the tag that will concatenated to all the resources' name tag.</td>
  </tr>
  <tr>
    <td>natInstanceType</td>
    <td>The instance type for the NAT Instance.</td>
  </tr>
  <tr>
    <td>natKeyName</td>
    <td>PEM/Key file used to launch the NAT instance. Do note that the PEM/Key file must be created before you can select it in this dropdown.</td>
  </tr>
  <tr>
    <td>natSshAccessCidr</td>
    <td>IP Address with CIDR notation, to secure Bastion Host only access to the NAT Instance.</td>
  </tr>
  <tr>
    <td>PemFileEC2Instances</td>
    <td>PEM/Key file that should be used to launch the EC2 instances.</td>
  </tr>
  <tr>
    <td>privateSubnet1Cidr</td>
    <td>CIDR for private subnet in primary AZ.</td>
  </tr>
  <tr>
    <td>privateSubnet2Cidr</td>
    <td>CIDR for private subnet in secondary AZ.</td>
  </tr>
  <tr>
    <td>publicSubnet1Cidr</td>
    <td>CIDR for public subnet in primary AZ.</td>
  </tr>
  <tr>
    <td>publicSubnet2Cidr</td>
    <td>CIDR for public subnet in secondary AZ.</td>
  </tr>
  <tr>
    <td>primaryAZ</td>
    <td>Primary AZ for this infrastructure. </td>
  </tr>
  <tr>
    <td>secondaryAZ</td>
    <td>Secondary AZ for this infrastructure used for High Availability.</td>
  </tr>
  <tr>
    <td>vpcCidr</td>
    <td>CIDR which will be used for addressing the VPC. This CIDR will be broken into four subnets.</td>
  </tr>
</table>

#### EC2 Instance Linux User and Password
For this evaluation, we have put a static password within the CloudFormation template.   
If needed, you can change the password for ubuntu user from within the CloudFormation template for each EC2 instance. You will have to change the password within the UserData key for EC2 instances by editing the following line where default password is *qFKWZRUFfk17BmD7BNSvx9O6* :

```"echo -e \"qFKWZRUFfk17BmD7BNSvx9O6\nqFKWZRUFfk17BmD7BNSvx9O6\n\" | passwd ubuntu\n",```

It is also needed that you replace the S3 bucket URL in the EC2 UserData key which hosts the different Chef-client configuration and certificate files for your Chef server.    
After successful deployment, go to the Outputs tab to find the private IP of the App servers and Web server along with the bastion host or jumpbox.

### B. Jenkins Build Plan Projects

#### Java Application Build Plan

The Application build plan automates the following tasks:
* Pulls source code from git repository.
* Runs various tests.
* Publishes test reports.
* Creates and publishes Docker image to Docker Hub.

To use the code provided by us, there are certain pre-requisites that need to be met. The following two plugins for Jenkins must be installed:

* Checkstyle Plug-in
* Cobertura Plugin

After importing *“java_application_buildplan_config.xml”* within Jenkins you will have to reconfigure certain configurations. The Configuration changes required are as follows:

* To trigger builds on commit pushed to a git repository, please configure webhooks for that repository where the ‘Hello World’ application code is uploaded.
* Add the repository URL and add git credentials under the source code management section within Jenkins. By default, we have configured our job to pull the master branch.
* Under the Build section, we have a script that prepares the docker image. Within that script you will have to make the following changes:

     ```sudo docker build -t inadevops/helloworld:$BUILD_NUMBER -t inadevops/helloworld:latest . ``` 

In the above code, please change the docker repository name from dockerhub’s inadevops repository to your docker repository.

* In the next line, replace INADEV’s repository credentials with your docker repository credentials.

     ```sudo docker login -u inadevops -p password```


* Lastly, replace *inadevops* with your repository name, wherever applicable.

#### Docker Image Deployment

The Docker image deployment plan automates the following tasks:
* Pulls source code from git repository for Chef cookbook for Docker.
* Logs into the Application servers using SSH through the bastion host.
* Executes *chef-client* command to pull the configuration changes from Chef server.

The Configuration changes required are as follows:

* Import *“docker_image_deployment_config.xml”*.
* To trigger builds on commit pushed to git repository, please configure webhooks for that repository where the Chef cookbook  uploaded, add the repository URL and add git credentials under the source code management section within Jenkins. By default, we have configured our job to pull the master branch.
* In the Build section, within the Shell script, you will have to replace the values of the variables bastion_host_ip, appserver1_ip and appserver2_ip with the IP addresses that will be published under the Outputs tab of the CloudFormation console, post successful deployment of infrastructure using the CloudFormation template.

*It is advisable not to change the password that is set against the variable serverpass. If you are required to change that, do make sure that the same password is set within the CloudFormation template.*

#### Web Server Configuration Deployment

The Web Server configuration deployment plan automates the following tasks:
* Pulls source code from git repository for Chef cookbook for Web Server.
* Logs into the Application servers using SSH through the bastion host.
* Executes *chef-client* command to pull the configuration changes from Chef server.

The Configuration changes required are as follows:

* Import “web_server_configuration_deployment_config.xml”.  
* To trigger builds on commit pushed to git repository, please configure webhooks for that repository where the Chef cookbook  uploaded, add the repository URL and add git credentials under the source code management section within Jenkins. By default, we have configured our job to pull the master branch.
* In the shell script section you will have to replace the values of variables webserver1_ip and webserver2_ip with the private IP addresses of the Web Servers along with value for variable bastion_host_ip which also can be found under the Outputs tab of the CloudFormation console.

### C. Chef Cookbooks

To version control the Cookbooks and for auto deployment, these cookbooks must be uploaded into a git repository. URL and credentials of these repositories must be specified within the Jenkins deployment plans for Docker and Web Server configuration. It is required that each cookbook is uploaded to its own separate repository.

#### Webserver Cookbook

Following tasks are performed by this cookbook:

* On first run, installs and configures Apache2 and its modules.
* Next run onwards it updates the Apache2 configuration files.

To evaluate our Chef cookbook for Web Server configuration, you will have to replace the value of the variable backendserverip with the IP address of the App Server within the file webserver/recipes/default.rb

#### Docker Cookbook

Following tasks are performed by this cookbook:

* Fetches the latest tag of the Docker image built by Jenkins from the repository.
* Stops and removes the old container if running.
* Deploys container in the App Server.

Following changes are required to evaluate the cookbook:
* Within the docker/recipes/default.rb file, please replace *“inadevops”* with your repository name and also do make sure that you have replaced our Docker Hub credentials with yours. Again, these changes are not necessary to evaluate this cookbook as you can use INADEV’s docker repository.
* If you decide to make the above-mentioned changes and use your own docker repository, it is further required that you change the repository name within a file located at docker/templates/docker-status.erb which is a shell script template used to check docker container status.

### D. Java Application

This is a Spring Boot Application which is deployed on Tomcat.

It is required that you push the “Hello World” application to a git repository.   
Do note that you will have to use this repository URL and credentials within Jenkins as mentioned in Java Application Build Plan.
