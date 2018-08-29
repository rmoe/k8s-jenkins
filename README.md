# Jenkins on IBM Cloud

## Steps

1. [Install Helm](#1-install-helm)
2. [Instal Jenkins](#2-install-helm)
3. [Log into Jenkins](#3-log-into-jenkins)
4. [Configure Jenkins](#4-configure-jenkins)
5. [Build Docker Images](#5-build-docker-images)

### 1. Install Helm
```bash
$ helm init
```

Running `helm ls` should execute without error. If you see an error that says
something like `Error: could not find a ready tiller pod` just wait a little
longer and try again.

### 2. Install Jenkins

Prior to installing Jenkins a persistent volume must be created.

```bash
$ kubectl apply -f volume.yaml`
```

Now Jenkins can be installed using helm.

```bash
$ helm install --name jenkins stable/jenkins --set rbac.install=true \
               --set Persistence.Enabled=true \
               --set Persistence.StorageClass=jenkins-pv
```

### 3. Log into Jenkins

First you need to get the generated admin password.
```bash
$ printf $(kubectl get secret --namespace default jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```

A load balancer is created for Jenkins. When that's ready you can log in with
username `admin` and the password from the previous step. Run the following
commands to determine the login URL for Jenkins.

```bash
$ export SERVICE_IP=$(kubectl get svc --namespace default jenkins --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")
$ echo http://$SERVICE_IP:8080/login

```

### 4. Configure Jenkins

#### Configure credentials

In order for Jenkins to be able to launch pods for running jobs you have
to configure the service account credentials.

Under "Manage Jenkins" > "Configure Jenkins" > "Cloud" > "Credentials"
select "Add".

![New credentials](images/credentials.png?raw=true)


#### Create a pipeline

Pipelines are used to model a build process. Agents are dynamically created
for each pipeline run. Those agents are a pod running one or more containers.
Steps in the pipeline can be run on any of the containers in the agent pod.

![Pipeline](images/new_pipeline.png?raw=true)

```groovy
pipeline {
  agent any

  stages {
    stage('Test') {
      steps {
        container('nodejs') {
          sh "node --version"
        }
      }
    }
  }
}
```

#### Add additional containers

Additional containers can be run in the agent pod. In the pipeline above
the first step of the 'Test' stage runs on a nodejs container.

![Additional container](images/new_container.png?raw=true)

### 5. Build Docker images
Docker images can be built as part of the pipeline. The docker socket
from the host needs to be shared with the agent containers by creating a
host path volume.

Volume configuration is under "Manage Jenkins" > "Configure Jenkins" > "Cloud".
![Docker volume](images/docker_volume.png?raw=true)

A new stage can then be added to the pipeline.

```groovy
stage('Build') {
  steps {
    container('build') {
        sh 'apk update && apk install docker'
        sh 'docker build -t application .'
      }
    }
  }
}
```

#### Push images to the IBM Container Registry

Automatically pushing images to the IBM container registry requires an API key.

```bash
$ ibmcloud iam api-key-create
```

This API key can be passed into the pipeline via environment variables.
In the container configuration add a new environment variable named
`REGISTRY_TOKEN`.

```groovy
stage('Build') {
  steps {
    container('build') {
        sh 'apk update && apk install docker'
        sh 'docker login -u token -p ${REGISTRY_TOKEN} ng.registry.bluemix.net'
        sh 'docker build -t application .'
        sh 'docker tag ${IMAGE_REPO}/application application'
      }
    }
  }
}
```
