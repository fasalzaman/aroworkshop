# LAB 2: ARO INTERNALS

## 2.1 Application Overview
### Resources

- The source code for this app is available here: <https://github.com/openshift-cs/ostoy>
- OSToy front-end container image: <https://quay.io/aroworkshop/ostoy-frontend>
- OSToy microservice container image: <https://quay.io/aroworkshop/ostoy-microservice>
- Deployment Definition YAMLs:
  - [ostoy-fe-deployment.yaml](/yaml/ostoy-fe-deployment.yaml)
  - [ostoy-microservice-deployment.yaml](/yaml/ostoy-microservice-deployment.yaml)

> **Note** In order to simplify the deployment of the app (which you will do next) we have included all the objects needed in the above YAMLs as "all-in-one" YAMLs.  In reality though, an enterprise would most likely want to have a different yaml file for each Kubernetes object.

### About OSToy

OSToy is a simple Node.js application that we will deploy to Azure Red Hat OpenShift. It is used to help us explore the functionality of Kubernetes. This application has a user interface which you can:

- write messages to the log (stdout / stderr)
- intentionally crash the application to view self-healing
- toggle a liveliness probe and monitor OpenShift behavior
- read config maps, secrets, and env variables
- if connected to shared storage, read and write files
- check network connectivity, intra-cluster DNS, and intra-communication with an included microservice

### OSToy Application Diagram

![OSToy Diagram](/media/managedlab/4-ostoy-arch.png)

### Familiarization with the Application UI

1. Shows the pod name that served your browser the page.
2. **Home:** The main page of the application where you can perform some of the functions listed which we will explore.
3. **Persistent Storage:**  Allows us to write data to the persistent volume bound to this application.
4. **Config Maps:**  Shows the contents of configmaps available to the application and the key:value pairs.
5. **Secrets:** Shows the contents of secrets available to the application and the key:value pairs.
6. **ENV Variables:** Shows the environment variables available to the application.
7. **Networking:** Tools to illustrate networking within the application.
8. Shows some more information about the application.

![Home Page](/media/managedlab/10-ostoy-homepage-1.png)

### Learn more about the application

To learn more, click on the "About" menu item on the left once we deploy the app.

![ostoy About](/media/managedlab/5-ostoy-about.png)

## 2.2 Application Deployment

### Retrieve login command

If not logged in via the CLI, click on the dropdown arrow next to your name in the top-right and select *Copy Login Command*.

![CLI Login](../media/managedlab/7-ostoy-login.png)

Then go to your terminal and paste that command and press enter.  You will see a similar confirmation message if you successfully logged in.

```sh
$ oc login https://openshift.abcd1234.eastus.azmosa.io --token=hUXXXXXX
Logged into "https://openshift.abcd1234.eastus.azmosa.io:443" as "okashi" using the token provided.

You have access to the following projects and can switch between them with 'oc project <projectname>':

    aro-demo
  * aro-shifty
  ...
```

### Create new project

Create a new project called "OSToy" in your cluster.

Use the following command

`oc new-project ostoy`

You should receive the following response

```sh
$ oc new-project ostoy
Now using project "ostoy" on server "https://openshift.abcd1234.eastus.azmosa.io:443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-25-centos7~https://github.com/sclorg/ruby-ex.git

to build a new example application in Ruby.
```

Equivalently you can also create this new project using the web UI by selecting "Application Console" at the top  then clicking on "+Create Project" button on the right.

![UI Create Project](../media/managedlab/6-ostoy-newproj.png)


### Download YAML configuration

Download the Kubernetes deployment object yamls from the following locations to your Azure Cloud Shell, in a directory of your choosing (just remember where you placed them for the next step).

Feel free to open them up and take a look at what we will be deploying. For simplicity of this lab we have placed all the Kubernetes objects we are deploying in one "all-in-one" yaml file.  Though in reality there are benefits to separating these out into individual yaml files.

[ostoy-fe-deployment.yaml](/yaml/ostoy-fe-deployment.yaml)

[ostoy-microservice-deployment.yaml](/yaml/ostoy-microservice-deployment.yaml)

### Deploy backend microservice

The microservice application serves internal web requests and returns a JSON object containing the current hostname and a randomly generated color string.


In your command line deploy the microservice using the following command:

`oc apply -f ostoy-microservice-deployment.yaml`

You should see the following response:
```
$ oc apply -f ostoy-microservice-deployment.yaml
deployment.apps/ostoy-microservice created
service/ostoy-microservice-svc created
```

### Deploy the front-end service

The frontend deployment contains the node.js frontend for our application along with a few other Kubernetes objects to illustrate examples.

 If you open the *ostoy-fe-deployment.yaml* you will see we are defining:

- Persistent Volume Claim
- Deployment Object
- Service
- Route
- Configmaps
- Secrets

In your command line deploy the frontend along with creating all objects mentioned above by entering:

`oc apply -f ostoy-fe-deployment.yaml`

You should see all objects created successfully

```sh
$ oc apply -f ostoy-fe-deployment.yaml
persistentvolumeclaim/ostoy-pvc created
deployment.apps/ostoy-frontend created
service/ostoy-frontend-svc created
route.route.openshift.io/ostoy-route created
configmap/ostoy-configmap-env created
secret/ostoy-secret-env created
configmap/ostoy-configmap-files created
secret/ostoy-secret created
```

### Get route

Get the route so that we can access the application via `oc get route`

You should see the following response:

```sh
NAME           HOST/PORT                                                      PATH      SERVICES              PORT      TERMINATION   WILDCARD
ostoy-route   ostoy-route-ostoy.apps.abcd1234.eastus.azmosa.io             ostoy-frontend-svc   <all>                   None
```

Copy `ostoy-route-ostoy.apps.abcd1234.eastus.azmosa.io` above and paste it into your browser and press enter.  You should see the homepage of our application.

![Home Page](../media/managedlab/10-ostoy-homepage.png)

## 2.3 Logging and Metrics
Assuming you can access the application via the Route provided and are still logged into the CLI (please go back to part 2 if you need to do any of those) we'll start to use this application.  As stated earlier, this application will allow you to "push the buttons" of OpenShift and see how it works.  We will do this to test the logs.

Click on the *Home* menu item and then click in the message box for "Log Message (stdout)" and write any message you want to output to the *stdout* stream.  You can try "**All is well!**".  Then click "Send Message".

![Logging stdout](../media/managedlab/8-ostoy-stdout.png)

Click in the message box for "Log Message (stderr)" and write any message you want to output to the *stderr* stream. You can try "**Oh no! Error!**".  Then click "Send Message".

![Logging stderr](../media/managedlab/9-ostoy-stderr.png)

### View logs directly from the pod

Go to the CLI and enter the following command to retrieve the name of your frontend pod which we will use to view the pod logs:

```sh
$ oc get pods -o name
pod/ostoy-frontend-679cb85695-5cn7x
pod/ostoy-microservice-86b4c6f559-p594d
```

So the pod name in this case is **ostoy-frontend-679cb85695-5cn7x**.  Then run `oc logs ostoy-frontend-679cb85695-5cn7x` and you should see your messages:

```sh
$ oc logs ostoy-frontend-679cb85695-5cn7x
[...]
ostoy-frontend-679cb85695-5cn7x: server starting on port 8080
Redirecting to /home
stdout: All is well!
stderr: Oh no! Error!
```

You should see both the *stdout* and *stderr* messages.

## 2.4 Exploring Health Checks

In this section we will intentionally crash our pods as well as make a pod non-responsive to the liveness probes and see how Kubernetes behaves.  We will first intentionally crash our pod and see that Kubernetes will self-heal by immediately spinning it back up. Then we will trigger the health check by stopping the response on the `/health` endpoint in our app. After three consecutive failures, Kubernetes should kill the pod and then recreate it.

It would be best to prepare by splitting your screen between the OpenShift Web UI and the OSToy application so that you can see the results of our actions immediately.

![Splitscreen](../media/managedlab/23-ostoy-splitscreen.png)

But if your screen is too small or that just won't work, then open the OSToy application in another tab so you can quickly switch to the OpenShift Web Console once you click the button. To get to this deployment in the OpenShift Web Console go to: 

*Applications > Deployments >* click the number in the "Last Version" column for the "ostoy-frontend" row

![Deploy Num](../media/managedlab/11-ostoy-deploynum.png)

Go to the OSToy app, click on *Home* in the left menu, and enter a message in the "Crash Pod" tile (ie: "This is goodbye!") and press the "Crash Pod" button.  This will cause the pod to crash and Kubernetes should restart the pod. After you press the button you will see:

![Crash Message](../media/managedlab/12-ostoy-crashmsg.png)

Quickly switch to the Deployment screen. You will see that the pod is red, meaning it is down but should quickly come back up and show blue.

![Pod Crash](../media/managedlab/13-ostoy-podcrash.png)

You can also check in the pod events and further verify that the container has crashed and been restarted.

![Pod Events](../media/managedlab/14-ostoy-podevents.png)

Keep the page from the pod events still open from the previous step.  Then in the OSToy app click on the "Toggle Health" button, in the "Toggle Health Status" tile.  You will see the "Current Health" switch to "I'm not feeling all that well".

![Pod Events](../media/managedlab/15-ostoy-togglehealth.png)

This will cause the app to stop responding with a "200 HTTP code". After 3 such consecutive failures ("A"), Kubernetes will kill the pod ("B") and restart it ("C"). Quickly switch back to the pod events tab and you will see that the liveness probe failed and the pod as being restarted.

![Pod Events2](../media/managedlab/16-ostoy-podevents2.png)

## 2.5 Persistent Storage
In this section we will execute a simple example of using persistent storage by creating a file that will be stored on a persistent volume in our cluster and then confirm that it will "persist" across pod failures and recreation.

Inside the OpenShift web UI click on *Storage* in the left menu. You will then see a list of all persistent volume claims that our application has made.  In this case there is just one called "ostoy-pvc".  You will also see other pertinent information such as whether it is bound or not, size, access mode and age.  

In this case the mode is RWO (Read-Write-Once) which means that the volume can only be mounted to one node, but the pod(s) can both read and write to that volume.  The default in ARO is for Persistent Volumes to be backed by Azure Disk, but it is possible to chose Azure Files so that you can use the RWX (Read-Write-Many) access mode.  ([See here for more info on access modes](https://docs.openshift.com/aro/architecture/additional_concepts/storage.html#pv-access-modes))

In the OSToy app click on *Persistent Storage* in the left menu.  In the "Filename" area enter a filename for the file you will create. (ie: "test-pv.txt")

Underneath that, in the "File Contents" box, enter text to be stored in the file. (ie: "Azure Red Hat OpenShift is the greatest thing since sliced bread!" or "test" :) ).  Then click "Create file".

![Create File](../media/managedlab/17-ostoy-createfile.png)

You will then see the file you created appear above under "Existing files".  Click on the file and you will see the filename and the contents you entered.

![View File](../media/managedlab/18-ostoy-viewfile.png)

We now want to kill the pod and ensure that the new pod that spins up will be able to see the file we created. Exactly like we did in the previous section. Click on *Home* in the left menu.

Click on the "Crash pod" button.  (You can enter a message if you'd like).

Click on *Persistent Storage* in the left menu

You will see the file you created is still there and you can open it to view its contents to confirm.

![Crash Message](../media/managedlab/19-ostoy-existingfile.png)

Now let's confirm that it's actually there by using the CLI and checking if it is available to the container.  If you remember we mounted the directory `/var/demo_files` to our PVC.  So get the name of your frontend pod

`oc get pods`

then get an SSH session into the container

`oc rsh <podname>`

then `cd /var/demo_files`

if you enter `ls` you can see all the files you created.  Next, let's open the file we created and see the contents

`cat test-pv.txt`

You should see the text you entered in the UI.

```
$ oc get pods
NAME                                  READY     STATUS    RESTARTS   AGE
ostoy-frontend-5fc8d486dc-wsw24       1/1       Running   0          18m
ostoy-microservice-6cf764974f-hx4qm   1/1       Running   0          18m

$ oc rsh ostoy-frontend-5fc8d486dc-wsw24
/ $ cd /var/demo_files/

/var/demo_files $ ls
lost+found   test-pv.txt

/var/demo_files $ cat test-pv.txt 
Azure Red Hat OpenShift is the greatest thing since sliced bread!
```

Then exit the SSH session by typing `exit`. You will then be in your CLI.

## 2.6 Configuration

In this section we'll take a look at how OSToy can be configured using [ConfigMaps](https://docs.openshift.com/container-platform/3.11/dev_guide/configmaps.html), [Secrets](https://docs.openshift.com/container-platform/3.11/dev_guide/secrets.html), and [Environment Variables](https://docs.openshift.com/container-platform/3.11/dev_guide/environment_variables.html).  This section won't go into details explaining each (the links above are for that), but will show you how they are exposed to the application.  

### Configuration using ConfigMaps

ConfigMaps allow you to decouple configuration artifacts from container image content to keep containerized applications portable.

Click on *Config Maps* in the left menu.

This will display the contents of the configmap available to the OSToy application.  We defined this in the `ostoy-fe-deployment.yaml` here:

```sh
kind: ConfigMap
apiVersion: v1
metadata:
  name: ostoy-configmap-files
data:
  config.json:  '{ "default": "123" }'
```

### Configuration using Secrets

Kubernetes Secret objects allow you to store and manage sensitive information, such as passwords, OAuth tokens, and ssh keys. Putting this information in a secret is safer and more flexible than putting it, verbatim, into a Pod definition or a container image.

Click on *Secrets* in the left menu.

This will display the contents of the secrets available to the OSToy application.  We defined this in the `ostoy-fe-deployment.yaml` here:

```sh
apiVersion: v1
kind: Secret
metadata:
  name: ostoy-secret
data:
  secret.txt: VVNFUk5BTUU9bXlfdXNlcgpQQVNTV09SRD1AT3RCbCVYQXAhIzYzMlk1RndDQE1UUWsKU01UUD1sb2NhbGhvc3QKU01UUF9QT1JUPTI1
type: Opaque
```

### Configuration using Environment Variables

Using environment variables is an easy way to change application behavior without requiring code changes. It allows different deployments of the same application to potentially behave differently based on the environment variables, and OpenShift makes it simple to set, view, and update environment variables for Pods/Deployments.

Click on *ENV Variables* in the left menu.

This will display the environment variables available to the OSToy application.  We added three as defined in the deployment spec of `ostoy-fe-deployment.yaml` here:

```sh
  env:
  - name: ENV_TOY_CONFIGMAP
    valueFrom:
      configMapKeyRef:
        name: ostoy-configmap-env
        key: ENV_TOY_CONFIGMAP
  - name: ENV_TOY_SECRET
    valueFrom:
      secretKeyRef:
        name: ostoy-secret-env
        key: ENV_TOY_SECRET
  - name: MICROSERVICE_NAME
    value: OSTOY_MICROSERVICE_SVC
```

The last one, `MICROSERVICE_NAME` is used for the intra-cluster communications between pods for this application.  The application looks for this environment variable to know how to access the microservice in order to get the colors.

## 2.7 Networking and Scaling

In this section we'll see how OSToy uses intra-cluster networking to separate functions by using microservices and visualize the scaling of pods.

Let's review how this application is set up...

![OSToy Diagram](../media/managedlab/4-ostoy-arch.png)

As can be seen in the image above we see we have defined at least 2 separate pods, each with its own service.  One is the frontend web application (with a service and a publicly accessible route) and the other is the backend microservice with a service object created so that the frontend pod can communicate with the microservice (across the pods if more than one).  Therefore this microservice is not accessible from outside this cluster, nor from other namespaces/projects (due to ARO's network policy, **ovs-networkpolicy**).  The sole purpose of this microservice is to serve internal web requests and return a JSON object containing the current hostname and a randomly generated color string.  This color string is used to display a box with that color displayed in the tile titled "Intra-cluster Communication".

### Networking

Click on *Networking* in the left menu. Review the networking configuration.

The right tile titled "Hostname Lookup" illustrates how the service name created for a pod can be used to translate into an internal ClusterIP address. Enter the name of the microservice following the format of `my-svc.my-namespace.svc.cluster.local` which we created in our `ostoy-microservice.yaml` which can be seen here:

```sh
apiVersion: v1
kind: Service
metadata:
  name: ostoy-microservice-svc
  labels:
    app: ostoy-microservice
spec:
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: ostoy-microservice
```

In this case we will enter: `ostoy-microservice-svc.ostoy.svc.cluster.local`

We will see an IP address returned.  In our example it is ```172.30.165.246```.  This is the intra-cluster IP address; only accessible from within the cluster.

![ostoy DNS](../media/managedlab/20-ostoy-dns.png)

### Scaling

OpenShift allows one to scale up/down the number of pods for each part of an application as needed.  This can be accomplished via changing our *replicaset/deployment* definition (declarative), by the command line (imperative), or via the web UI (imperative). In our deployment definition (part of our `ostoy-fe-deployment.yaml`) we stated that we only want one pod for our microservice to start with. This means that the Kubernetes Replication Controller will always strive to keep one pod alive.

If we look at the tile on the left we should see one box randomly changing colors.  This box displays the randomly generated color sent to the frontend by our microservice along with the pod name that sent it. Since we see only one box that means there is only one microservice pod.  We will now scale up our microservice pods and will see the number of boxes change.

To confirm that we only have one pod running for our microservice, run the following command, or use the web UI.

```sh
[okashi@ok-vm ostoy]# oc get pods
NAME                                   READY     STATUS    RESTARTS   AGE
ostoy-frontend-679cb85695-5cn7x       1/1       Running   0          1h
ostoy-microservice-86b4c6f559-p594d   1/1       Running   0          1h
```

Let's change our microservice definition yaml to reflect that we want 3 pods instead of the one we see.  Download the [ostoy-microservice-deployment.yaml](/yaml/ostoy-microservice-deployment.yaml) and save it on your local machine.

Open the file using your favorite editor. Ex: `vi ostoy-microservice-deployment.yaml`.

Find the line that states `replicas: 1` and change that to `replicas: 3`. Then save and quit.

It will look like this

```sh
spec:
    selector:
      matchLabels:
        app: ostoy-microservice
    replicas: 3
 ```

Assuming you are still logged in via the CLI, execute the following command:

`oc apply -f ostoy-microservice-deployment.yaml`

Confirm that there are now 3 pods via the CLI (`oc get pods`) or the web UI (*Overview > expand "ostoy-microservice"*).

See this visually by visiting the OSToy app and seeing how many boxes you now see.  It should be three.

![UI Scale](../media/managedlab/22-ostoy-colorspods.png)

Now we will scale the pods down using the command line.  Execute the following command from the CLI: 

`oc scale deployment ostoy-microservice --replicas=2`

Confirm that there are indeed 2 pods, via the CLI (`oc get pods`) or the web UI.

See this visually by visiting the OSToy App and seeing how many boxes you now see.  It should be two.

Lastly let's use the web UI to scale back down to one pod.  In the project you created for this app (ie: "ostoy") in the left menu click *Overview > expand "ostoy-microservice"*.  On the right you will see a blue circle with the number 2 in the middle. Click on the down arrow to the right of that to scale the number of pods down to 1.

![UI Scale](../media/managedlab/21-ostoy-uiscale.png)

See this visually by visiting the OSToy app and seeing how many boxes you now see.  It should be one.  You can also confirm this via the CLI or the web UI


## 2.8 Autoscaling

In this section we will explore how the [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) (HPA) can be used and works within Kubernetes/OpenShift. 

As defined in the Kubernetes documentation:
> Horizontal Pod Autoscaler automatically scales the number of pods in a replication controller, deployment, replica set or stateful set based on observed CPU utilization.

We will create an HPA and then use OSToy to generate CPU intensive workloads.  We will then observe how the HPA will scale up the number of pods in order to handle the increased workloads.  

#### 1. Create the Horizontal Pod Autoscaler

Run the following command to create the autoscaler. This will create an HPA that maintains between 1 and 10 replicas of the Pods controlled by the *ostoy-microservice* DeploymentConfig created. Roughly speaking, the HPA will increase and decrease the number of replicas (via the deployment) to maintain an average CPU utilization across all Pods of 80% (since each pod requests 50 millicores, this means average CPU usage of 40 millicores)

`oc autoscale deployment/ostoy-microservice --cpu-percent=80 --min=1 --max=10`

#### 2. View the current number of pods

In the OSToy app in the left menu click on "Autoscaling" to access this portion of the workshop.  

![HPA Menu](../media/managedlab/32-hpa-menu.png)

As was in the networking section you will see the total number of pods available for the microservice by counting the number of colored boxes.  In this case we have only one.  This can be verified through the web UI or from the CLI.

You can use the following command to see the running microservice pods only:
`oc get pods --field-selector=status.phase=Running | grep microservice`

![HPA Main](../media/managedlab/33-hpa-mainpage.png)

#### 3. Increase the load

Now that we know that we only have one pod let's increase the workload that the pod needs to perform. Click the link in the center of the card that says "increase the load".  **Please click only *ONCE*!**

This will generate some CPU intensive calculations.  (If you are curious about what it is doing you can click [here](https://github.com/openshift-cs/ostoy/blob/master/microservice/app.js#L32)).

> **Note:** The page may become slightly unresponsive.  This is normal; so be patient while the new pods spin up.

#### 4. See the pods scale up

After about a minute the new pods will show up on the page (represented by the colored rectangles). Confirm that the pods did indeed scale up through the OpenShift Web Console or the CLI (you can use the command above).

> **Note:** The page may still lag a bit which is normal.

#### 5. Review resources in Azure Monitor

After confirming that the autoscaler did spin up new pods, revisit Azure Monitor like we did in the logging section.  By clickin on the containers tab we can see the resource consumption of the pods and see that three pods were created to handle the load.

![HPA Metrics](../media/managedlab/34-ostoy-hpametrics.png)
