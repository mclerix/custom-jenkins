# custom-jenkins
Repository that explains how to create a custom jenkins image for Openshift (3.4) using Docker Build, S2I and Persistent volumes<br>
**OpenShift Jenkins Image Repository:** https://github.com/openshift/jenkins

## Dockerfile
### Using Jenkins Template provided by OpenShift
Jenkins is customisable using **docker build** (example in the directory Dockerfile). To do that, do the following steps:
1. Create a Dockerfile that extends the original openshift3/jenkins-2-rhel7
```Dockerfile
FROM openshift/jenkins-2-rhel7
COPY plugins.txt /opt/openshift/configuration/plugins.txt
RUN /usr/local/bin/install-plugins.sh /opt/openshift/configuration/plugins.txt
```
2. Build the image using the Dockerfile above
```bash
docker build -t docker.io/clrxm/custom-jenkins:latest .
```
3. Push the image in a docker registry
```
docker push docker.io/clrxm/custom-jenkins:latest
```
4. In OpenShift, create a new project
```
oc new-project custom-jenkins --display-name="Dockerfile Jenkins" --description="Demonstrate custom jenkins image with docker build"
```
5. Create an imageStream with your image (This will import all the tags).
```
oc import-image custom-jenkins --from=docker.io/clrxm/custom-jenkins --confirm --all
```
*You can repeat this command to import new tags*
6. Deploy Jenkins using the OpenShift jenkins template
```
oc new-app jenkins-ephemeral -p NAMESPACE=custom-jenkins -p JENKINS_IMAGE_STREAM_TAG=custom-jenkins:latest
```

### Using custom template
1. Repeat steps 1, 2, & 3 above.
1. In OpenShift, create a new project
```
oc new-project custom-jenkins --display-name="Dockerfile Jenkins" --description="Demonstrate custom jenkins image with docker build"
```
1. Import the custom template under directory Dockerfile
```
oc create -f https://raw.githubusercontent.com/clerixmaxime/custom-jenkins/Dockerfile/jenkins-docker-image-template.yml
```
1. Deploy Jenkins. You have to specify the JENKINS_IMAGE to use as environement variable
```
oc new-app jenkins-master-s2i -p JENKINS_IMAGE=docker.io/clrxm/custom-jenkins:latest
```

## S2I
You can use the S2I feature of OpenShift to build new Custom Jenkins Images.
In order to include your modifications in Jenkins image, you need to have a Git repository with following directory structure:

  * `./plugins` folder that contains binary Jenkins plugins you want to copy into Jenkins
  * `./plugins.txt` file that list the plugins you want to install (see the section above)
  * `./configuration/jobs` folder that contains the Jenkins job definitions
  * `./configuration/config.xml` file that contains your custom Jenkins configuration

Note that the `./configuration` folder will be copied into `/var/lib/jenkins` folder, so you can also include additional files (like `credentials.xml`, etc.).

1. Create a new project.
```
oc new-project custom-jenkins-s2i
```
1. Build the new custom-jenkins image using S2I jenkins image provided by OpenShift.
```
oc new-build jenkins:2~https://github.com/clerixmaxime/custom-jenkins.git --context-dir=/s2i --name=custom-jenkins
```
1. Deploy Jenkins using the OpenShift jenkins template
```
oc new-app jenkins-ephemeral \
     -p NAMESPACE=custom-jenkins-s2i  -p JENKINS_IMAGE_STREAM_TAG=custom-jenkins:latest
```

## Latest Openshift Jenkins Image / Custom template
Building the last version of the openshift/jenkins image, an environment variable INSTALL_PLUGINS has been integrated that allows to provide a list of plugins. The image will install all these plugins at start up.

1. Create a new project.
```
oc new-project custom-jenkins-image
```
1. Import the image in Openshift as an ImageStream **(The image has already been built and is available on my Docker Hub. Refer to ttps://github.com/openshift/jenkins to build the image on your own or retag my image and add it to your own registry)**
```
oc import-image jenkins --from=docker.io/clrxm/jenkins-2-centos7 --confirm
```
1. Download the template `jenkins-ephemeral-latest.json` under s2i
1. Import the template
```
oc create -f jenkins-ephemeral-latest.json
```
1. Deploy Jenkins using this template
```
oc new-app jenkins-ephemeral-latest -p NAMESPACE=custom-jenkins-image -p JENKINS_IMAGE_STREAM_TAG=jenkins:latest -p INSTALL_PLUGINS=blueocean:1.0.1
```

## persistentVolumes
You can also create a custom-jenkins image by mounting a persistent volume with the required plugins.
1. Create a Persistent Volume on OpenShift (Depends on your backing-storage solution)
1. Copy the required files (`/plugins` directory) in the persistent volume <br>
> **/!\ The `/plugins` directory must contain all plugin dependencies**

1. Create a new project.
```
oc new-project custom-jenkins-pv
```
1. Deploy Jenkins using the OpenShift jenkins-persistent template
```
oc new-app jenkins-persistent
```

## storageClasses
