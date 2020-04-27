# helm-helloworld
# demo-1

This readme provides a detailed, simple example of using a Helm Chart in the SaaS DevOps CD pipeline. A simple Hello World web application is used to demonstrate how to install a chart to your OpenShift cluster and then walkthrough various lifecycle stages of a Helm Chart such as upgrade, rollback and delete.

Watch the demo recording - [click here](https://drive.google.com/open?id=1SjoKBY8xlUPKzBXPKvMAxNdP_IL4F_Rh)

**Note, ignore the commentary in the demo recording about needing to use the "helm-pipeline" string in the commit message in order to trigger the pipeline for helm. This is no longer needed and the default for ANY commit message to the repo is to trigger the pipeline. If you wanted to do a commit but don't want to invoke the command in the helm-command.yml file, for example you just had an edit to the README, use the "skip cd pipeline" string in the commmit message instead.**

Suggest allocating 30-60 minutes to go through the steps listed below.

For a very quick introduction to see how Helm is supported in the SaaS DevOps CD pipeline, about 15 minutes, either follow the [Helm Kubernetes Dashboard example](https://github.gwd.broadcom.net/dockcpdev/helm-kubernetes-dashboard) or the [Saas DevOps teams sample app](https://github.gwd.broadcom.net/dockcpdev/helm-sampleapp). There are additional examples you can follow at the bottom of this README.

For a more general overview of the SaaS DevOps CD pipeline support for Helm, see the [recording](https://drive.google.com/open?id=11EI4lqr-IklNwdNZY8HofWaOm_R4-GKK) done for the Automic team on May 29th:

* General Overview - 1 through 15 minutes
* Demo of deploying an external Helm Chart (Kubernetes Dashboard) - 15 through 38 minutes
* Demo of deploying an internal Helm Chart (Helloworld) - 38 minutes through 52 minutes
* Summing up - 52 minutes through 55 minutes

New to Helm? Assuming you understand Kubernetes and OpenShift basics, we suggest reviewing the following content available freely on the internet:

* [Helm - Quick Start Guide](https://helm.sh/docs/using_helm/#quickstart-guide)
* [Helm - Documentation](https://helm.sh/docs/)
* [Helm - Github](https://github.com/helm/helm)
* [Helm - Developing Charts](https://helm.sh/docs/developing_charts/)
* [Bitnami - How to Create your first Helm Chart](https://docs.bitnami.com/kubernetes/how-to/create-your-first-helm-chart/)
* [Codefresh - Helm Best Practices](https://codefresh.io/docs/docs/new-helm/helm-best-practices/)
* [IBM - Create a Helm Package (chart)](https://www.ibm.com/support/knowledgecenter/en/SS8TQM_1.1.0/app_center/add_package.html)
* [Banzai Cloud - Creating Helm Charts](https://banzaicloud.com/blog/creating-helm-charts/)

## Overview

The Hello World Helm Chart compressed tarball used in this README  is accessible from the development Helm repository in artifactory at:

```
https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local/helm-helloworld/helloworld-0.1.tgz
```

The Hello World docker image used in the chart provdies a simple web page, showing an image, and is available from the developemnt Docker registry in artifactory at:

```
https://artifactory-lvn.broadcom.net/artifactory/docker-release-candidate-local/helm-helloworld/helloworld
```

To demonstrate Helm upgrades and rollbacks, there are three helloworld images tagged as demo1, demo2 and demo3, each showing a different image when deployed.

The user experience in running a Helm command in the Saas DevOps CD pipeline is similar to anyone familiar with using the compose or openshift-template entrypoint options in the pipeline. To use Helm in the pipeline, define ```entrypoint``` in [deploy-info.yml](./deploy-info.yml) as ```helm```. For example:

```
project_name: "XXXXXXXX"
email: "XXXXXXXX@broadcom.com"
version: "1.0"
distribute_to_bintray: "yes"
entrypoint: "helm"
openshift:
  env:
    name: "XXXXXXXX"
    url: "XXXXXXXX"
    username: "XXXXXXXX"
    password: "XXXXXXXX"
flowdock: 
  flow_token: "XXXXXXXX"    
image_scan:
  skip_image_scan: "no"
  severity_threshold: 'warn'
  ignore_failures: 'yes'
```

The installation and subsequent lifecycle actions of the Helm chart is triggered via settings in a [helm-command.yml](./helm-command.yml) file in the local repo directory, with custom helm settings in a [values.yaml](./values.yaml), as per the usual Helm functionality. The Jenkinsfile pipeline that invokes the Helm call is triggered by committing any changes made to ```helm-command.yml``` and/or ```values.yaml``` to your Github application repo.

The Helm command to be actioned is defined in ```helm-command.yaml``` and it also contains an images section where you would list any docker images used by ```values.yaml```, in the main chart or any subcharts. For a Helm install or upgrade, those docker images are pushed through the pipeline to the Openshift docker registry, prior to any Helm Chart install/upgrade command running.

The  Helm pipeline firstly promotes any docker images defined in ```helm-command.yml``` to your OpenShift registry if the action is install or upgrade. It then processes the defined Helm command. Given that any docker images defined in ```helm-command.yml``` will have been promoted to your local OpenShift registry by the Helm pipeline prior to when the Helm command is invoked, the image(s) referenced by the Helm Chart in ```values.yaml``` need to refer to the local OpenShift docker registry (e.g. ```docker-registry.default.svc:5000/<project-name><image-name>:<image tag>```). See the helm-helloworld ```values.yaml``` file for an example of a ```values.yaml``` image and tag setting referring to the local OpenShift registry. Note, the use of the Saas DevOps CD pipeline to manage the promotion of your docker images referenced in ```values.yaml``` is strongly encouraged but not enforced. You can for example, maybe for development or testing purposes, ignore or remove the images section from ```helm-command.yaml``` and have any ```values.yaml``` image references refer directly to docker images in a public docker registry (e.g. docker hub) as long as that docker registry is (internet) accessible from your OpenShift cluster.

The following fields in ```helm-command.yml``` defines the Helm command that will be actioned to your cluster when the Helm pipeline is triggered. The (Jenkins) Helm pipeline is triggered whenever a commit is made to the your git application repository. Note, to make a commit to your application repository and not trigger the Helm pipelein, include the string ```skip pipeline``` in the git message. You may want to do that, for example, if making a README update only.

* **repo**: Url to the Helm repo that contains the Chart.
* **username**: Artifactory username to authenticate to the repo, if required.
* **password**: Artifactory API Key (or password) to authenticate to the repo, if required.
* **chart**: Folder path and name of the Helm chart within the repo.
* **values**: Path, including filename, to ```values.yaml```
* **command**: The Helm command will be invoked by the pipeline. See below for supported values.
* **commandOptions**: Any additonal arguments to be passed into the command.
* **releaseName**: Used by some Helm calls. See command below.
* **releaseNumber**: Used by some Helm calls. See command below.
* **images**: A list of docker images to be pushed through the pipeline prior to a Helm install/upgrade. Note, for any other Helm action, any image names listed are ignored. Also, although not recommended, if your chart refers to some other docker registry and the OpenShift cluster has (internet) access to that docker registry, then you can remove the ```images``` section totaly and have the image values defined in ```values.yaml``` point directly at that docker registry. In doing so, you would not be taking advantage of the inherent image scanning and any other controls that the SaaS DevOps pipeline provides, but for general development purposes trying out charts available in public Helm repositories, might be something you could try in a non-production environment.

An example ```helm-command.yml``` for a Helm install command, with debug output, for a chart named helloworld taken from the local Helm artifactory repo is listed below. The Helm chart refers to a single docker image which we push through the pipeline to be available in the local OpenShift docker registry. If the repo required username/password authentication, you can obtain your Artifactory API Key from viewing your User Profile in the Jfrog Artifactory UI.

```
repo: https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: install
commandOptions: --debug
images:
  helloworld: docker-release-candidate-local.artifactory-lvn.broadcom.net/helm-helloworld/helloworld:demo1
```

Supported values for **command** are:

* **[install](https://helm.sh/docs/helm/#helm-install)**: Installs a Helm chart release to the project using the provided ```values.yaml```.
* **[list](https://helm.sh/docs/helm/#helm-list)**: List all Helm chart releases.
* **[status](https://helm.sh/docs/helm/#helm-status)**: Get the status of a Helm chart release.
* **[get](https://helm.sh/docs/helm/#helm-get)**: Get the details of a Helm chart release.
* **[test](https://helm.sh/docs/helm/#helm-test)**: Run the tests defined in a Helm chart release.
* **[get-hooks](https://helm.sh/docs/helm/#helm-get-hooks)**: Get the hooks of a Helm chart release.
* **[get-manifest](https://helm.sh/docs/helm/#helm-get-manifest)**: Get the openshift yaml manifest of a Helm chart release.
* **[get-notes](https://helm.sh/docs/helm/#helm-get-notes)**: Get the NOTES.txt from a Helm chart release.
* **[get-values](https://helm.sh/docs/helm/#helm-get-values)**: Get the ```values.yaml``` used for a Helm chart release.
* **[history](https://helm.sh/docs/helm/#helm-history)**: Get the history of a Helm chart release.
* **[upgrade](https://helm.sh/docs/helm/#helm-upgrade)**: Upgrade a Helm chart release using the provided ```values.yaml```.
* **[rollback](https://helm.sh/docs/helm/#helm-rollback)**: Rollback a Helm chart release to the provided release number.
* **[delete](https://helm.sh/docs/helm/#helm-delete)**: Delete a Helm chart release.
* **[version](https://helm.sh/docs/helm/#helm-version)**: Returns the version information of the Helm client and (tiller) server.
* **[reset](https://helm.sh/docs/helm/#helm-reset)**: Removes the helm server (tiller) from the cluster. WARNING, only do this if you are sure you want to wipe away Helm totally. If there are Helm releases deployed, you would need to use the --force flag.

## The Demo

### Prerequesites

**Please use the script below to create your own (private) copy of any of the provided Helm examples, and do not edit the content of this repo directly please!**

For Mac or Linux desktop, from a terminal command prompt, download [helm_demo_setup.sh](https://drive.google.com/open?id=1LPFR88rSZuyqucdysEehJpz2SPYsWXWW) and ensure it is executable.
For Microsoft Windows 10 or later, enable the ability to execute bash shell scripts by [installing the Windows Subsystem Linux](http://techgenix.com/bash-on-windows-10/) then open a (bash) terminal command prompt, download [helm_demo_setup.sh](https://drive.google.com/open?id=1LPFR88rSZuyqucdysEehJpz2SPYsWXWW) and ensure it is executable.

Run the ```helm_demo_setup.sh``` executable from your desktop and provide input to the prompts when directed. For example:

```
$ ./helm_demo_setup.sh 
Please enter your Broadcom Okta ID: tr658049
Please enter the Broadcom IT dockcpdev repo to clone: helm-helloworld
Creating private repo in dockcpdev named helm-helloworld-tr658049 - please provide your github password
Enter host password for user 'tr658049':
...
Compressing objects: 100% (31/31), done.
Writing objects: 100% (32/32), 734.44 KiB | 28.25 MiB/s, done.
Total 32 (delta 4), reused 0 (delta 0)
remote: Resolving deltas: 100% (4/4), done.
To https://github.gwd.broadcom.net/dockcpdev/helm-helloworld-tr658049
 * [new branch]      master -> master
Branch 'master' set up to track remote branch 'master' from 'origin'.
cd into helm-helloworld-tr658049 and follow the README instructions for your helm demo
```

You should then be able to cd into the new repo folder and work with this local copy of the repo, commiting changes and triggering the pipeline for your new repo.

Edit the ```deploy-info.yml``` to refer to your OpenShift cluster. Remember to set the entrypoint value to ```helm``` and suggest setting ```project_name``` to the value of the repo you have just created. For example:

```
project_name: "helm-kubernetes-dashboard-tr658049"
email: "tommy.reilly@broadcom.com"
version: "1.0"
distribute_to_bintray: "yes"
entrypoint: "helm"
openshift:
  env:
    name: "gdue4"
flowdock: 
  flow_token: "50f1839f1a5552dfe3d2785978868ace"    
image_scan:
  skip_image_scan: "no"
  severity_threshold: "warn"
  ignore_failures: "yes"
```

Edit the Jenkinsfile to trigger the pipeline on commit of your repo. The dockcpdev organisation includes an implicit hook that will run the pipeline for your repo when there is a commit when the following Jenkins file is found in the top level of your repo:

```
@Library('casaas-tools-pipeline-library@master') _
node {
	runDefaultCDPipeline {}
}
```

In order to automatically create an OpenShift router, edit the ```openshift``` section at the top of ```values.yaml```, ensuring the ```enabled``` setting is set ```true``` and define the ```applicationHostname``` setting to the base path to your OpenShift applications. To control the manner that the route handles traffic, add appropriate annotations as defined [here](https://docs.okd.io/3.11/architecture/networking/routes.html#route-specific-annotations). For example, to define a whitelist of IP addresses that can access the router add the ```haproxy.router.openshift.io/ip_whitelist``` annotation. In order to define the load-balancing characteristics of the router, add the `haproxy.router.openshift.io/balance` annotation and set it to either `source`, roundrobin` or `leastconn`.

```
openshift:
  enabled: true
  # Application Hostname: The exposed hostname that will be used for the route to the helloworld service.
  applicationHostname: app.gdue4.saasdev.broadcom.com
  # If the router is for *.app, set router to "app". If the router is for *.infra, set router to "infra"
  router: app
  annotations: {}
    # Remove the annotation brackets, uncomment the haproxy.router.openshift.io/iop_whitelist annotation line below and list appropriate IP addresses, in order to limit access
    # to the route to a defined list of ip address ranges
    # See https://docs.openshift.com/container-platform/3.11/architecture/networking/routes.html#whitelist for the format of the IP address list (space delimited)
    # Default Broadcom IP network address ranges include 192.19.0.0/16 202.86.248.0/24 202.153.6.0/24 130.119.0.0/16 130.200.0.0/16 138.42.0.0/16 141.202.0.0/16 155.35.0.0/16
    # Reach out to Broadcom IT (e.g. via a 1.Support ticket) if you need to identify any other IP address ranges
    # haproxy.router.openshift.io/ip_whitelist: <REPLACE WITH IP ADDRESSES>
    # To define load-balancing behaviour, uncomment out the following and set it to either source, roundrobin or leastconn
    # haproxy.router.openshift.io/balance: roundrobin
```

Alternatively, you can set the openshift enabled value to false and manually create a router via the OpenShift UI to expose the helm-helloworld service.

### Step 1 - Install the Hello World demo1 release

Edit ```helm-command.yml``` as follows:

```
repo: https:/artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: install
commandOptions: --debug
images:
  helloworld: docker-release-candidate-local.artifactory-lvn.broadcom.net/helm-helloworld/helloworld:demo1
```

and edit ```values.yaml``` as follows:

```
image:
  repository: docker-registry.default.svc:5000/<YOUR_PROJECT_NAME>/helloworld
  tag: demo1
  pullPolicy: IfNotPresent
```

Replace <YOUR_PROJECT_NAME> with the value of the ```project_name``` field defined in your ```deploy-info.yml```.

If this is the first time you run the Helm pipeline, the Helm server (tiller) will be installed to your cluster. The tiller service account and tiller deployment will be added to the tiller namespace (created if it doesn't exist) with cluster-admin access.

Merge the changes above to your repo. The Jenkins pipeline will be triggered and the Hello World chart will be deployed to your cluster.

Review the Jenkins log output. You will see detail of your Helm deployment, especially if you defined ```--debug``` for commandOptions. Identify the name of your Helm release - in our example it is ```lumpy-mouse```:

```
NAME: lumpy-mouse
REVISION: 1
RELEASED: Fri Apr 26 23:28:20 2019
CHART: helloworld-0.1
USER-SUPPLIED VALUES:
```

You can get the route url either from the OpenShift console or from the OpenShift command line. The NOTES.txt has a helpful section, displayed in the Jenkins log output, that you can copy and paste to a command line to identify the URL:

```
NOTES:
1. Get the application URL by running these commands:
  From a command line, login to OpenShift using the oc client (you can get the Login Command via the OpenShift UI) and run the following commands:
  export ROUTE=$(oc get routes --namespace helm-helloworld -l "app.kubernetes.io/name=helloworld,app.kubernetes.io/instance=lumpy-mouse" -o jsonpath="{.items[0].spec.host}")
  echo "Visit https://$ROUTE to use your application"
```

If you browse to the url defined by the route you should see the following:

![Alt text](images/screenshot1.png?raw=true "Hello World for demo1")

### Step 2 - Upgrade the Hello World release from demo1 to demo2 and then to demo3

Edit ```helm-command.yml``` defining:

```
repo: https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: upgrade
commandOptions: --debug
releaseName: lumpy-mouse
images:
  helloworld: docker-release-candidate-local.artifactory-lvn.broadcom.net/helm-helloworld/helloworld:demo2
```

and edit ```values.yaml``` defining:

```
image:
  repository: docker-registry.default.svc:5000/helm-helloworld/helloworld
  tag: demo2
  pullPolicy: IfNotPresent
```

Merge the changes above to your repo. The Jenkins pipeline will be triggered and the Hello World chart will be upgraded to your cluster.

If you browse to the url defined by the route, you should see the following:

![Alt text](images/screenshot2.png?raw=true "Hello World for demo2")

Note, you will likely need to open a new Incognito/Private window to see the updated release version as the local browser cache may, by default, show the prior version.

Let's do one more upgrade to demo3. Follow the same steps as above but change the image tag field in ```values.yaml``` to demo3 and the image tag for the helloworld image in the images section of ```helm-command.yml``` to demo3.

![Alt text](images/screenshot3.png?raw=true "Hello World for demo3")

### Step 3 - List the Helm releases

Edit ```helm-action.yml``` as follows:

```
repo: https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: list
commandOptions: --debug
```

After merging the changes above to github, the Jenkins pipeline will be triggered and the Hello World chart call will be made to your cluster. If you browse the Jenkins logfile output for your pipeline run, you should see something like the following:

```
NAME          	REVISION	UPDATED                 	STATUS  	CHART         	APP VERSION	NAMESPACE      
lumpy-mouse	3       	Mon Apr 29 21:55:03 2019	DEPLOYED	helloworld-0.1	1.0        	helm-helloworld
```

### Step 4 - See the history of the Helm release

Edit ```helm-action.yml``` as follows:

```
repo: https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: history
commandOptions: --debug
releaseName: lumpy-mouse
```

After merging the changes above to github, the Jenkins pipeline will be triggered and the Hello World chart call will be made to your cluster. If you browse the Jenkins logfile output for your pipeline run, you should see something like the following:

```
REVISION	UPDATED                 	STATUS  	CHART         	DESCRIPTION      
1       	Fri Apr 26 23:28:20 2019	SUPERSEDED	helloworld-0.1	Install complete
2       	Mon Apr 29 21:46:29 2019	SUPERSEDED	helloworld-0.1	Upgrade complete
3       	Mon Apr 29 21:55:03 2019	DEPLOYED	helloworld-0.1	Upgrade complete
```

### Step 5 - Rollback the Hello World release from demo3 to demo1

Edit ```helm-command.yml``` as follows:

```
repo: https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: rollback
commandOptions: --debug
releaseName: lumpy-mouse
releaseNumber: 1
```

The ```releaseNumber``` can be found in the history output from Step 4 (REVISION field).

Merge the changes above to your repo. The Jenkins pipeline will be triggered and the Hello World chart will be rolled back to demo1 in your cluster.

If you browse to the url defined by the route, you should see the following:

![Alt text](images/screenshot1.png?raw=true "Rollback the Hello World release from demo3 to demo1")

Note, you may need to open a new Incognito/Private window to see the updated release version as the local browser cache may, by default, show the prior version.

### Step 6 - Show the history of the Hello World release after after the rollback

Invoke the pipeline again as per Step 4 and you should see output similar to the following:

```
REVISION	UPDATED                 	STATUS    	CHART         	DESCRIPTION     
1       	Mon Apr 15 21:46:29 2019	SUPERSEDED	helloworld-0.1	Install complete
2       	Mon Apr 15 22:29:59 2019	SUPERSEDED	helloworld-0.1	Upgrade complete
3       	Mon Apr 29 21:55:03 2019	SUPERSEDED	helloworld-0.1	Upgrade complete
4       	Mon Apr 30 08:41:43 2019	DEPLOYED  	helloworld-0.1	Rollback to 1 
```

### Step 7 - Delete (purge) the Hello World release

Edit ```helm-command.yml``` as follows:

```
repo: https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: delete
commandOptions: --debug --purge
releaseName: lumpy-mouse
```

After merging the changes above to github, the Jenkins pipeline will be triggered and the Hello World chart call will be made to your cluster. If you browse the Jenkins logfile output for your pipeline run, you should see a message stating the release has been deleted. You can also confirm via the the OpenShift UI or ```oc``` CLI that the deployment no longer exists in your namespace.

## Try out the other Helm commands

You only need to update the ```helm-command.yml``` to action the following Helm commands. See the Jenkins log output for the return values.

### get

```
repo: https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: get
commandOptions: --debug
```

### test

```
repo: https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: test
commandOptions: --debug
```

### get-hooks

```
repo: https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: get-hooks
commandOptions: --debug
```

### get-manifest

```
repo: https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: get-manifest
commandOptions: --debug
```

### get-notes

```
repo: https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: get-notes
commandOptions: --debug
```

### get-values

```
repo: https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: get-values
commandOptions: --debug
```

### version

```
repo: https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: version
commandOptions: --debug
```

### reset

```
repo: https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local
chart: helm-helloworld/helloworld-0.1.tgz
values: ./values.yaml
command: reset
commandOptions: --debug --force
```

## What Next?

Check out any of the following additional examples.

* [helm-sampleapp](https://github.gwd.broadcom.net/dockcpdev/helm-sampleapp) - a simple example
* [helm-kubernetes-dashboard](https://github.gwd.broadcom.net/dockcpdev/helm-kubernetes-dashboard) - an example pulling images from a public registry and then the pipeline
* [helm-ghost](https://github.gwd.broadcom.net/dockcpdev/helm-ghost) - an example showing how [dependencies](https://github.com/helm/helm/blob/master/docs/helm/helm_dependency.md) are managed by Helm
* [helm-mysql](https://github.gwd.broadcom.net/dockcpdev/helm-mysql) - an example pulling image(s) through the pipeline
* [helm-postgresql](https://github.gwd.broadcom.net/dockcpdev/helm-postgresql) - an example pulling image(s) through the pipeline
* [helm-openshift-template](https://github.gwd.broadcom.net/dockcpdev/helm-openshift-template) - an example using native OpenShift objects built from a public Github
* [helm-kubeapps](https://github.gwd.broadcom.net/dockcpdev/helm-kubeapps) - a detailed example pulling image(s) through the pipeline

Try out creating your own Helm chart (or download one from the various public repositories) and running it via the Helm pipeline.

Note, in order to push a Helm packaged chart (i.e. in tgz format) to Artifactory, you need to use cURL. For example, replacing XXXXXXX with your Artifactory API Key, pushing a chart named my-chart with version number 0.1 to the Helm development repository:

```
curl -H 'X-JFrog-Art-Api:'XXXXXXXX'' -T my-chart-0.1.tgz https://artifactory-lvn.broadcom.net/artifactory/helm-release-candidate-local/my-chart/my-chart-0.1.tgz
```

## Authors

* **Tommy Reilly** - *Initial work*
