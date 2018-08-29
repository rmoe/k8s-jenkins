# Jenkins on IBM Cloud

## Installing Jenkins with Helm

### Installing Helm
```bash
$ helm init
```

### Installing Jenkins

Before Jenkins can be installed a persistent volume must be created.

```bash
$ kubectl apply -f volume.yaml`
```

Now Jenkins can be installed using helm.

```bash
$ helm install --name jenkins stable/jenkins --set rbac.install=true \
               --set Persistence.Enabled=true \
               --set Persistence.StorageClass=jenkins-pv
```

Once that's finished the admin password can be retrieved:

```bash
$ printf $(kubectl get secret --namespace default jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```

A load balancer is created for Jenkins. When that's ready you can log in with
username `admin` and the password from the previous step.

```bash
$ export SERVICE_IP=$(kubectl get svc --namespace default jenkins --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")
$ echo http://$SERVICE_IP:8080/login

```

### Configuring Jenkins

In order for Jenkins to be able to launch pods for running jobs you have
to configure the service account credentials.

Under "Manage Jenkins" > "Configure Jenkins" > "Cloud" > "Credentials"
select "Add".

![New credentials](images/credentials.png?raw=true)


#### Creating a pipeline

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


#### Adding additional containers

Additional containers can be run ion the agent pod. In the pipeline above
the first step of the 'Test' stage runs on a nodejs container.

![Additional container](images/new_container.png?raw=true)

### Building Docker images
Docker images can be built as part of the build pipeline.

#### Pushing new images to IBM Container Registry

