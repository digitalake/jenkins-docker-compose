# Jenkins-docker-compose

>You can find jenkins repo by github link [here](https://github.com/digitalake/jenkins-test)

### Step 1 — Disabling the Setup Wizard

Using JCasC eliminates the need to show the setup wizard; therefore, in this first step, you’ll create a modified version of the official [jenkins/jenkins](https://hub.docker.com/r/jenkins/jenkins) image that has the setup wizard disabled. You will do this by creating a Dockerfile and building a custom Jenkins image from it.

The __jenkins/jenkins__ image allows you to enable or disable the setup wizard by passing in a system property named __jenkins.install.runSetupWizard__ via the __JAVA_OPTS__ environment variable. Users of the image can pass in the __JAVA_OPTS__ environment variable at runtime using the __--env__ flag to __docker run__. However, this approach would put the onus of disabling the setup wizard on the user of the image. Instead, you should disable the setup wizard at build time, so that the setup wizard is disabled by default.

You can achieve this by creating a __Dockerfile__ and using the __ENV__ instruction to set the __JAVA_OPTS__ environment variable.

Example:

 <img src="https://user-images.githubusercontent.com/109740456/213938831-c712dc8a-0e31-4295-a5bf-8d17ada57cd4.png" width="600">

### Step 2 — Installing Jenkins Plugins

To use JCasC, you need to install the Configuration as Code plugin. Currently, no plugins are installed. You can confirm this by navigating to __http://server_ip:8080/pluginManager/installed__

In this step, you’re going to modify your Dockerfile to pre-install a selection of plugins, including the Configuration as Code plugin.

To automate the plugin installation process, you can make use of an installation script that comes with the __jenkins/jenkins__ Docker image. You can use __jenkins-plugin-cli__. To use it, you would need to:

  - Create a text file containing a list of plugins to install or declare plugins right in dockerfile
  - Copy the file into the Docker image(if using text file)
  - Run the jenkins-plugin-cli script to install the plugins
  
 For executing without file:
 ```
 RUN jenkins-plugin-cli --plugins "plugin1 plugin2 plugin3"
 ```
 
 For executing with file:
 ```
 RUN jenkins-plugin-cli -f /path/to/file
 ```
 
 Example:
 
 <img src="https://user-images.githubusercontent.com/109740456/213939250-506e09f7-4d24-4f3c-8494-db40c0b3d8c5.png" width="400">
 
The list contains the __Configuration as Code plugin__, as well as all the plugins suggested by the setup wizard (correct as of Jenkins v2.251). For example, you have the __Git plugin__, which allows Jenkins to work with Git repositories; you also have the __Pipeline plugin__, which is actually a suite of plugins that allows you to define Jenkins jobs as code.

After the Jenkins is fully up and running message appears on stdout, navigate to __server_ip:8080/pluginManager/installed__ to see a list of installed plugins. You will see a solid checkbox next to all the plugins you’ve specified inside __plugins.txt__ or by using inline list, as well as a faded checkbox next to plugins, which are dependencies of those plugins.

### Step 3 — Specifying the Jenkins URL

he Jenkins URL is a URL for the Jenkins instance that is routable from the devices that need to access it. For example, if you’re deploying Jenkins as a node inside a private network, the Jenkins URL may be a private IP address, or a DNS name that is resolvable using a private DNS server. For this tutorial, it is sufficient to use the server’s IP address (or __127.0.0.1 for local hosts__) to form the Jenkins URL.

You can set the Jenkins URL on the web interface by navigating to __server_ip:8080/configure__ and entering the value in the Jenkins URL field under the Jenkins Location heading. Here’s how to achieve the same using the Configuration as Code plugin:
  - Define the desired configuration of your Jenkins instance inside a declarative configuration file (which we’ll call __casc.yaml__).
  - Copy the configuration file into the Docker image.
  - Set the __CASC_JENKINS_CONFIG__ environment variable to the path of the configuration file to instruct the __Configuration as Code plugin__ to read it.
  
  Url section in __casc.yaml__:
  
  <img src="https://user-images.githubusercontent.com/109740456/213939986-8ee9e282-1c42-4335-878e-219b46b2b9ba.png" width="400">
  
  You can provide docker image with __CASC_JENKINS_CONFIG__ variable by using:
  ```
  ENV CASC_JENKINS_CONFIG /var/jenkins_home/casc.yaml
  ```
  
### Step 4 — Creating a User

So far, your setup has not implemented any authentication and authorization mechanisms. In this step, you will set up a basic, password-based authentication scheme and create a new user named admin.

Start by opening your __casc.yaml__ file

Then, add in the snippet:
```
jenkins:
  securityRealm:
    local:
      allowsSignup: false
      users:
       - id: ${JENKINS_ADMIN}
         password: ${JENKINS_ADMIN_PASSWORD}
```
In the context of Jenkins, a security realm is simply an authentication mechanism; the local security realm means to use basic authentication where users must specify their ID/username and password. Other security realms exist and are provided by plugins.

>Note that you’ve also specified __allowsSignup: false__, which prevents anonymous users from creating an account through the web interface.

Finally, instead of hard-coding the __user ID and password__, you are using variables whose values can be filled in at runtime. This is important because one of the benefits of using JCasC is that the __casc.yaml__ file can be committed into source control; if you were to store user passwords in plaintext inside the configuration file, you would have effectively compromised the credentials. Instead, variables are defined using the ${VARIABLE_NAME} syntax, and its value can be filled in using an environment variable of the same name, or you can add __.env__ file and declare variables there.

### Step 5 — Setting Up Authorization

After setting up the security realm, you must now configure the authorization strategy. In this step, you will use the __Matrix Authorization Strategy__ plugin to configure permissions for your admin user.

> __loggedInUsersCanDoAnything__: anonymous users are given either no access or read-only access. Authenticated users have full permissions to do everything. By allowing actions only for authenticated users, you are able to have an audit trail of which users performed which actions.

Add this code snippet to __casc.yaml__ file under users section:
```
...
  authorizationStrategy:
      globalMatrix:
        permissions:
          - "Overall/Administer:admin"
          - "Overall/Read:authenticated"
...
```
The globalMatrix property sets global permissions (as opposed to per-project permissions). The permissions property is a list of strings with the format __<permission-group>/<permission-name>:<role>__. Here, you are granting the __Overall/Administer__ permissions to the __admin user__. You’re also granting __Overall/Read__ permissions to __authenticated__, which is a special role that represents all authenticated users. There’s another special role called anonymous, which groups all non-authenticated users together. But since permissions are denied by default, if you don’t want to give anonymous users any permissions, you don’t need to explicitly include an entry for it.

### Step 6 — Setting Up Build Authorization

The first issue in the notifications list relates to build authentication. By default, all jobs are run as the system user, which has a lot of system privileges. Therefore, a Jenkins user can perform privilege escalation simply by defining and running a malicious job or pipeline; this is insecure.

Instead, jobs should be ran using the same Jenkins user that configured or triggered it. To achieve this, you need to install an additional plugin called the __Authorize Project__ plugin.

Add this code snippet to casc.yaml file as another section:
```
...
security:
  queueItemAuthenticator:
    authenticators:
    - global:
        strategy: triggeringUsersAuthorizationStrategy
...
```

So the __casc.yaml__ looks like this:

<img src="https://user-images.githubusercontent.com/109740456/213941027-e9b008f7-9e36-4c8a-86c3-745cc024ae8b.png" width="600">

The dockerfile:

<img src="https://user-images.githubusercontent.com/109740456/213941114-8652fe8c-59a7-4491-818e-685625861db1.png" width="600">

### Docker-compose

Docker-compose snippet:

```
---
 version: '3.8'
 services:
    jenkins:
      build:
        context: ./dockerfile
        dockerfile: Jenkins-master-Dockerfile
      restart: on-failure
      ports:
       - 8080:8080
      container_name: ${CONTAINER_NAME}
      environment:
        - JENKINS_ADMIN=${JENKINS_ADMIN}
        - JENKINS_ADMIN_PASSWORD=${JENKINS_ADMIN_PASSWORD}
      volumes:
        - jenkins-data:/var/jenkins_home
      networks:
        - jenkins
 volumes:
  jenkins-data:
    driver: local
 networks: 
  jenkins:
```
Where:
  - version - version of docker-compose ure using
  - services - declaring services 
  - jenkins - service jenkins
  - build - declaring dockerfile context for build and custom dockerfile name
  - restart - restart policy
  - ports - exposing ports
  - container_name - the name the container will get
  - environment - passing variables to fill __casc.yaml__ config
  - volumes - mapping volumes to local fs
  - networks - creating jenkins net (for example for adding runner-containers in future)
  
### How to use?

Steps:
  - make sure docker and docker-compose are present on your machine
  - clone the repo
  - add your variables to .env file
  - cd to the repo directory
  - execute the docker-compose command as shown below
  
  ```
  docker-compose --env-file ./compose_conf/.env up -d 
  ```
  


















