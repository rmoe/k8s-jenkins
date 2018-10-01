# Jenkins on IBM Cloud

In this Code Pattern we will install and configure Jenkins on IBM Kubernetes Service.
Helm will be used to install Jenkins which will then be set up to run CI pipelines
on the same Kubernetes cluster.

The pipeline created will run a test stage followed by a build stage if the test stage
was successful. After both of those stages complete successfully the container built
in the build stage will be pushed to the IBM container registry.

## Steps

1. [Prerequisites](#1-prerequisites)
2. [Install Helm](#2-install-helm)
3. [Install Jenkins](#3-install-jenkins)
4. [Log into Jenkins](#4-log-into-jenkins)
5. [Configure Jenkins](#5-configure-jenkins)
6. [Build Docker Images](#6-build-docker-images)


### 1. Prerequisites

- A Kubernetes cluster which can be created by following the IBM Kubernetes Service
installation [instrucions](https://console.bluemix.net/docs/containers/container_index.html#clusters).

- Once the Kubernetes cluster is ready follow these [instructions](https://github.com/helm/helm#install) to install the Helm client.

### 2. Install Helm

Now that the Helm client is installed it's time to install the server-side component
of Helm called Tiller. Tiller is what the Helm client talks to and it runs inside of
your cluster managing the chart installions. For more information on Helm you can check
out this Helm 101 [material](https://github.com/IBM/helm101).


```bash
$ helm init
```

Running `helm ls` should execute without error. If you see an error that says
something like `Error: could not find a ready tiller pod` just wait a little
longer and try again.

### 3. Install Jenkins

Prior to installing Jenkins a persistent volume must be created.

```bash
$ kubectl apply -f volume.yaml
```

Now Jenkins can be installed using the Helm chart in the stable repository. This is the default
Helm repository. Specifically this [chart](https://github.com/helm/charts/tree/master/stable/jenkins) will be installed.

This chart has a number of configurable parameters. For this installation the following
parameters need to be configured:

- `rbac.install` - Setting this to `true` creates a service account and ClusterRoleBinding which
is necessary for Jenkins to create pods.
- `Persistence.Enabled` - Enables persistence of Jenkins data using a PVC.
- `Persistence.StorageClass` - When the PVC is created it will request a volume of the specified
class. In this case it is set to `jenkins-pv` which is the `storageClassName` of the volume created
previously. Setting this to the same value as the class name from `volume.yaml` ensures that
Jenkins will use the persistent volume already created.

```bash
$ helm install --name jenkins stable/jenkins --set rbac.install=true \
               --set Persistence.Enabled=true \
               --set Persistence.StorageClass=jenkins-pv
```

### 4. Log into Jenkins

As part of the chart installation a random password is generated and a
Kubernetes secret is created. A Kubernetes secret is an object that contains
sensitive data such as a password or a token. Each item in a secret must be
base64 encoded. This secret contains a data item named 'jenkins-admin-password'
which must be decoded.

The following command gets the of that data item from the secret named 'jenkins'
and decodes the result.

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

### 5. Configure Jenkins

#### Configure credentials

In order for Jenkins to be able to launch pods for running jobs you have
to configure the service account credentials.

Under "Manage Jenkins" > "Configure Jenkins" > "Cloud" > "Credentials"
select "Add".

![New credentials](images/credentials.png?raw=true)

#### Configure containers

By default the agent pod contains just one container, the jnlp-slave. However,
you are not restricted to running everything on that container. You can add
any number of containers to the agent pod. For this example pipeline you'll
need to add a NodeJS container. This is configured in "Manage Jenkins" >
"Configure Jenkins" > "Cloud".

![NodeJS container](images/new_container.png?raw=true)

#### Create a pipeline

Pipelines are used to model a build process. Agents are dynamically created
for each pipeline run. Those agents are a pod running one or more containers.
Steps in the pipeline can be run on any of the containers in the agent pod.

First, create a new pipeline item.

![Pipeline](images/new_pipeline.png?raw=true)

Then select 'Pipeline script' and use the following pipeline.

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

This simple pipeline only has one stage named 'Test'. That stage has only one
step which just executes a single command. The key part of the step definition
is the `container('nodejs')` statement. This tells jenkins to run the step on the
container named 'nodejs' which was configured in the previous step. Each item
that is prefixed with `sh` will be executed in a new shell. In a real pipeline
this is where you'd do actual work like running unit tests.


#### Add additional containers

Different steps of the pipeline can run on different containers. This allows
you to do things like run tests for different parts of a codebase in
language-specific containers. Because the stages execute sequentially you could
also have a 'deploy' stage that runs after tests pass to deploy your application.
It could be a kubectl pod to deploy to Kubernetes, or a helm pod to update an
existing chart.

### 6. Build Docker images
Docker images can be built as part of the pipeline. Create another container
named 'build' using the alpine image.
The docker socket from the host also needs to be shared with the agent
containers by creating a host path volume.

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
`REGISTRY_TOKEN`. Update the build stage to log into the registry and
push the image.

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
