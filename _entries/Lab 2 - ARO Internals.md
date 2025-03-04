# Lab 2 - ARO Internals

### Estimated Duration: 120 minutes

Azure Red Hat OpenShift (ARO) is a managed service on Azure for container orchestration. It supports automated updates, secure networking, and persistent storage. The OSToy app, used in the workshop, is deployed within this environment, showcasing ARO's capabilities.

## Lab Objectives

You will be able to complete the following tasks:

- Task 1: Application Overview
- Task 2: Application Deployment
- Task 3: Logging and Metrics
- Task 4: Exploring Health Checks
- Task 5: Persistent Storage
- Task 6: Configuration
- Task 7: Networking and Scaling
- Task 8: Pod Autoscaling
- Task 9: Managing Worker Nodes
- Task 10: Azure Service Operator - Blob Store

## Task 1: Application Overview

### Resources

- The source code for this app is available here: <https://github.com/openshift-cs/ostoy>
- OSToy front-end container image: <https://quay.io/repository/ostoylab/ostoy-frontend?tab=tags>
- OSToy microservice container image: <https://quay.io/repository/ostoylab/ostoy-microservice?tab=tags>
- Deployment Definition YAMLs:
  - [ostoy-frontend-deployment.yaml](https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/ostoy-frontend-deployment.yaml)
  - [ostoy-microservice-deployment.yaml](https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/ostoy-microservice-deployment.yaml)

> **Note** In order to simplify the deployment of the app (which you will do next) we have included all the objects needed in the above YAMLs as "all-in-one" YAMLs.  In reality though, an enterprise would most likely want to have a different yaml file for each Kubernetes object.

### About OSToy

OSToy is a simple Node.js application that we will deploy to Azure Red Hat OpenShift. It is used to help us explore the functionality of Kubernetes. This application has a user interface which you can:

- write messages to the log (stdout / stderr)
- intentionally crash the application to view self-healing
- toggle a liveness probe and monitor OpenShift behavior
- read config maps, secrets, and env variables
- if connected to shared storage, read and write files
- check network connectivity, intra-cluster DNS, and intra-communication with an included microservice
- increase the load to view automatic scaling of the pods to handle the load (via the Horizontal Pod Autoscaler)

### OSToy Application Diagram

![OSToy Diagram](../media/managedlab/4-ostoy-arch.png)

### Familiarization with the Application UI

1. Shows the pod name that served your browser the page.
2. **Home:** The main page of the application where you can perform some of the functions listed which we will explore.
3. **Persistent Storage:**  Allows us to write data to the persistent volume bound to this application.
4. **Config Maps:**  Shows the contents of configmaps available to the application and the key:value pairs.
5. **Secrets:** Shows the contents of secrets available to the application and the key:value pairs.
6. **ENV Variables:** Shows the environment variables available to the application.
7. **Networking:** Tools to illustrate networking within the application.
8. **Pod Auto Scaling:** Explore the Horizontal Pod Autoscaler to see how increased loads are handled.
9. **About:** Shows some more information about the application.

![Home Page](../media/managedlab/lab1-task8-4.png)

### Learn more about the application

To learn more, click on the "About" menu item on the left once we deploy the app.

![ostoy About](../media/managedlab/5-ostoy-about.png)

## Task 2: Application Deployment

### Retrieve login command

1. If not logged in via the CLI, click on the dropdown arrow next to your name in the top-right and select *Copy Login Command*.

   ![CLI Login](../media/managedlab/7-ostoy-login.png)

1. A new tab will open click "Display Token"

1. Copy the command under where it says, "Log in with this token". Then go to your terminal and paste that command and press enter. You will see a similar confirmation message if you successfully logged in.

   ```
   $ oc login --token=sha256~qWBXdQ_X_4wWZor0XZO00ZZXXXXXXXXXXXX --server=https://api.abcs1234.westus.aroapp.io:6443
   Logged into "https://api.abcd1234.westus.aroapp.io:6443" as "kube:admin" using the token provided.

   You have access to 67 projects, the list has been suppressed. You can list all projects with 'oc projects'

   Using project "default".
   ```

### Create new project

1.  To Create a new project called "OSToy" in your cluster, Use the following command:

   ```
   oc new-project ostoy
   ```

1. You should receive the following response

   ```
   $ oc new-project ostoy
   Now using project "ostoy" on server "https://api.abcd1234.westus2.aroapp.io:6443".
   [...]
   ```

1. Equivalently you can also create this new project using the web console by selecting *Home > Projects* on the left menu, then clicking on the "Create Project" button on the right.

   ![UI Create Project](../media/managedlab/6-ostoy-newproj.png)

### View the YAML deployment objects

View the Kubernetes deployment object YAMLs.  If you wish you can download them from the following locations to your Azure Cloud Shell, to your local machine, or just use the direct link in the next steps.

Feel free to open them up and take a look at what we will be deploying. For simplicity of this lab, we have placed all the Kubernetes objects we are deploying in one "all-in-one" YAML file.  Though in reality there are benefits (ease of maintenance and less risk) to separating these out into individual files.

[ostoy-frontend-deployment.yaml](https://github.com/microsoft/aroworkshop/blob/master/yaml/ostoy-frontend-deployment.yaml)

[ostoy-microservice-deployment.yaml](https://github.com/microsoft/aroworkshop/blob/master/yaml/ostoy-microservice-deployment.yaml)

### Deploy backend microservice

1. The microservice serves internal web requests and returns a JSON object containing the current hostname and a randomly generated color string.

1. In your terminal deploy the microservice using the following command:

   ```
   oc apply -f https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/ostoy-microservice-deployment.yaml
   ```

1. You should see the following response:
   ```
   $ oc apply -f https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/ostoy-microservice-deployment.yaml
   deployment.apps/ostoy-microservice created
   service/ostoy-microservice-svc created
   ```

### Deploy the front-end service

This deployment contains the node.js frontend for our application along with a few other Kubernetes objects.

If you open the *ostoy-frontend-deployment.yaml* you will see we are defining:

- Persistent Volume Claim
- Deployment Object
- Service
- Route
- Configmaps
- Secrets

1. Deploy the frontend along with creating all objects mentioned above by entering:

   ```
   oc apply -f https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/ostoy-frontend-deployment.yaml
   ```

1. You should see all objects created successfully

   ```
   $ oc apply -f https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/ostoy-frontend-deployment.yaml
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

1. Get the route so that we can access the application via:

   ```
   oc get route
   ```
 
1. You should see the following response:

   ```
   NAME           HOST/PORT                                                      PATH      SERVICES              PORT      TERMINATION   WILDCARD
   ostoy-route   ostoy-route-ostoy.apps.abcd1234.westus2.aroapp.io             ostoy-frontend-svc   <all>                   None
   ```

1. Copy the link under **HOST/PORT** and paste it into your browser (use **http://**) and press enter. You should see the homepage of our application. 

   ![Home Page](../media/managedlab/10-ostoy-homepage.png)

## Task 3: Logging and Metrics

Assuming you can access the application via the Route provided and are still logged into the CLI (please go back to part 2 if you need to do any of those) we'll start to use this application.  As stated earlier, this application will allow you to "push the buttons" of OpenShift and see how it works.  We will do this to test the logs.

1. Click on the *Home* menu item and then click in the message box for "Log Message (stdout)" and write any message you want to output to the *stdout* stream.  You can try "**All is well!**".  Then click "Send Message".

   ![Logging stdout](../media/managedlab/8-ostoy-stdout.png)

1. Click in the message box for "Log Message (stderr)" and write any message you want to output to the *stderr* stream. You can try "**Oh no! Error!**".  Then click "Send Message".

   ![Logging stderr](../media/managedlab/9-ostoy-stderr.png)

### View logs directly from the pod

1. Go to the Azure CLI and enter the following command to retrieve the name of your frontend pod which we will use to view the pod logs:

   ```
   oc get pods -o name
   ```

1. You should see similar response:
   ```
   $ oc get pods -o name
   pod/ostoy-frontend-679cb85695-5cn7x
   pod/ostoy-microservice-86b4c6f559-p594d
   ```

1. So, the pod name in this case is **ostoy-frontend-679cb85695-5cn7x**.  Then run `oc logs ostoy-frontend-679cb85695-5cn7x` and you should see your messages:

1. You should see similar response:
   ```
   $ oc logs ostoy-frontend-679cb85695-5cn7x
   [...]
   ostoy-frontend-679cb85695-5cn7x: server starting on port 8080
   Redirecting to /home
   stdout: All is well!
   stderr: Oh no! Error!
   ```

1. You should see both the *stdout* and *stderr* messages.

1. Try to see them from within the OpenShift Web Console as well. Make sure you are in the "ostoy" project. In the left menu click *Workloads > Pods > \<frontend-pod-name>*.  Then click the "Logs" sub-tab.

   ![web-pods](../media/managedlab/9-ostoy-wclogs.png)

## Task 4: Exploring Health Checks

In this section we will intentionally crash our pods and also make a pod non-responsive to the liveness probes and see how Kubernetes behaves.  We will first intentionally crash our pod and see that Kubernetes will self-heal by immediately spinning it back up. Then we will trigger the health check by stopping the response on the `/health` endpoint in our app. After three consecutive failures, Kubernetes should kill the pod and then recreate it.

1. It would be best to prepare by splitting your screen between the OpenShift Web Console and the OSToy application so that you can see the results of our actions immediately.

   ![Splitscreen](../media/managedlab/23-ostoy-splitscreen.png)

1. But if your screen is too small or that just won't work, then open the OSToy application in another tab so you can quickly switch to the OpenShift Web Console once you click the button. To get to this deployment in the OpenShift Web Console go to the left menu and click: **Workloads > Deployments > "ostoy-frontend"**

1. Go to the browser tab that has your OSToy app, click on *Home* in the left menu, and enter a message in the "Crash Pod" tile (e.g., "This is goodbye!") and press the "Crash Pod" button.  This will cause the pod to crash and Kubernetes should restart the pod. After you press the button you will see:

   ![Crash Message](../media/managedlab/12-ostoy-crashmsg.png)

1. Quickly switch to the tab with the deployment showing in the web console. You will see that the pod turns yellowish, meaning it is down but should quickly come back up and show blue.  It does happen quickly so you might miss it.

   ![Pod Crash](../media/managedlab/13-ostoy-podcrash.gif)

1. You can also check in the pod events and further verify that the container has crashed and been restarted. Click on **Pods > [Pod Name] > Events**

   ![Pods](../media/managedlab/13.1-ostoy-fepod.png)
   
   ![Pod Events](../media/managedlab/14-ostoy-podevents.png)

1. Keep the page from the pod events still open from the previous step.  Then in the OSToy app click on the "Toggle Health" button, in the "Toggle health status" tile.  You will see the "Current Health" switch to "I'm not feeling all that well".

   ![Pod Events](../media/managedlab/15-ostoy-togglehealth.png)

1. This will cause the app to stop responding with a "200 HTTP code". After 3 such consecutive failures ("A"), Kubernetes will kill the pod ("B") and restart it ("C"). Quickly switch back to the pod events tab and you will see that the liveness probe failed and the pod as being restarted.

   ![Pod Events2](../media/managedlab/16-ostoy-podevents2.png)

## Task 5: Persistent Storage

In this section we will execute a simple example of using persistent storage by creating a file that will be stored on a persistent volume in our cluster and then confirm that it will "persist" across pod failures and recreation.

1. Inside the OpenShift web console click on *Storage > Persistent Volume Claims* in the left menu. You will then see a list of all persistent volume claims that our application has made.  In this case there is just one called "ostoy-pvc".  If you click on it you will also see other pertinent information such as whether it is bound or not, size, access mode and creation time.  

1. In this case the mode is RWO (Read-Write-Once) which means that the volume can only be mounted to one node, but the pod(s) can both read and write to that volume.  The [default in ARO](https://docs.microsoft.com/en-us/azure/openshift/openshift-faq#can-we-choose-any-persistent-storage-solution--like-ocs) is for Persistent Volumes to be backed by Azure Disk, but it is possible to [use Azure Files](https://docs.openshift.com/container-platform/latest/storage/persistent_storage/persistent-storage-azure-file.html) so that you can use the RWX (Read-Write-Many) access mode.  See here for more info on [access modes](https://docs.openshift.com/container-platform/latest/storage/understanding-persistent-storage.html#pv-access-modes_understanding-persistent-storage).

1. In the OSToy app click on *Persistent Storage* in the left menu.  In the "Filename" area enter a filename for the file you will create (e.g., "test-pv.txt"). Use the ".txt" extension so you can easily open it in the browser.

1. Underneath that, in the "File contents" box, enter text to be stored in the file. (e.g., "Azure Red Hat OpenShift is the greatest thing since sliced bread!"). Then click "Create file".

   ![](../media/Redhat-image16.png)

1. You will then see the file you created appear above under "Existing files".  Click on the file and you will see the filename and the contents you entered.

   ![](../media/Redhat-image17.png)

1. We now want to kill the pod and ensure that the new pod that spins up will be able to see the file we created. Exactly like we did in the previous section. Click on *Home* in the left menu.

1. Click on the "Crash pod" button.  (You can enter a message if you'd like).

1. Click on *Persistent Storage* in the left menu.

1. You will see the file you created is still there and you can open it to view its contents to confirm.

   ![Crash Message](../media/managedlab/19-ostoy-existingfile.png)

1. Now let's confirm that it's actually there by using the CLI and checking if it is available to the container.  If you remember we [mounted the directory](https://github.com/microsoft/aroworkshop/blob/master/yaml/ostoy-fe-deployment.yaml#L50) `/var/demo_files` to our PVC.  So get the name of your frontend pod:

   ```
   oc get pods
   ```

1. Then get an SSH session into the container

   ```
   oc rsh <pod name>
   ```

1. Then `cd /var/demo_files`

1. If you enter `ls` you can see all the files you created.  Next, let's open the file we created and see the contents by the following command: 

   ```
   cat test-pv.txt
   ```

1. You should see the text you entered in the UI.
   ```
   /var/demo_files $ cat test-pv.txt
   Azure Red Hat OpenShift is the greatest thing since sliced bread!
   ```

1. The output will be similar to the following:

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

1. Then exit the SSH session by typing `exit`. You will then be in your CLI.

## Task 6: Configuration

In this section we'll take a look at how OSToy can be configured using [ConfigMaps](https://docs.openshift.com/container-platform/latest/nodes/pods/nodes-pods-configmaps.html), [Secrets](https://docs.openshift.com/container-platform/latest/cicd/builds/creating-build-inputs.html#builds-input-secrets-configmaps_creating-build-inputs), and [Environment Variables](https://docs.openshift.com/container-platform/3.11/dev_guide/environment_variables.html).  This section won't go into details explaining each (the links above are for that), but will show you how they are exposed to the application.  

### Configuration using ConfigMaps

ConfigMaps allow you to decouple configuration artifacts from container image content to keep containerized applications portable.

1. In the OSToy app click on *Config Maps* in the left menu.

   ![Home Page](../media/managedlab/configmaps.png)

1. This will display the contents of the configmap available to the OSToy application.  We defined this in the `ostoy-fe-deployment.yaml` here:

   ```
   kind: ConfigMap
   apiVersion: v1
   metadata:
   name: ostoy-configmap-files
   data:
   config.json:  '{ "default": "123" }'
   ```

### Configuration using Secrets

Kubernetes Secret objects allow you to store and manage sensitive information, such as passwords, OAuth tokens, and ssh keys. Putting this information in a secret is safer and more flexible than putting it, verbatim, into a pod definition or a container image.

1. In the OSToy app click on *Secrets* in the left menu.

   ![Home Page](../media/managedlab/secrets.png)
1. This will display the contents of the secrets available to the OSToy application.  We defined this in the `ostoy-fe-deployment.yaml` here:

   ```
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

1. In the OSToy app click on *ENV Variables* in the left menu.

   ![Home Page](../media/managedlab/env-variables.png)
1. This will display the environment variables available to the OSToy application.  We added three as defined in the deployment spec of `ostoy-fe-deployment.yaml` here:

   ```
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

1. The last one, `MICROSERVICE_NAME` is used for the intra-cluster communications between pods for this application.  The application looks for this environment variable to know how to access the microservice in order to get the colors.

## Task 7: Networking and Scaling

In this section we'll see how OSToy uses intra-cluster networking to separate functions by using microservices and visualize the scaling of pods.

1. Let's review how this application is set up...

   ![OSToy Diagram](../media/managedlab/4-ostoy-arch.png)

   As can be seen in the image above we have defined at least 2 separate pods, each with its own service.  One is the frontend web application (with a service and a publicly accessible route) and the other is the backend microservice with a service object created so that the frontend pod can communicate with the microservice (across the pods if more than one).  Therefore this microservice is not accessible from outside this cluster (or from other namespaces/projects, if configured, due to OpenShifts' network policy, [ovs-networkpolicy](https://docs.openshift.com/container-platform/latest/networking/network_policy/about-network-policy.html#nw-networkpolicy-about_about-network-policy)).  The sole purpose of this microservice is to serve internal web requests and return a JSON object containing the current hostname and a randomly generated color string.  This color string is used to display a box with that color displayed in the tile titled "Intra-cluster Communication".

### Networking
1. In the OSToy app click on *Networking* in the left menu. Review the networking configuration.

      ![Home Page](../media/managedlab/networking.png)
1. The right tile titled "Hostname Lookup" illustrates how the service name created for a pod can be used to translate into an internal ClusterIP address. Enter the name of the microservice following the format of `my-svc.my-namespace.svc.cluster.local` which we created in our `ostoy-microservice.yaml` as seen here:

   ```
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

1. In this case we will enter: `ostoy-microservice-svc.ostoy.svc.cluster.local` in the hostname parameter, and click on **DNS Lookup**.

1. We will see an IP address returned.  In our example it is `172.30.165.246`.  This is the intra-cluster IP address; only accessible from within the cluster.

   ![ostoy DNS](../media/managedlab/20-ostoy-dns.png)

### Scaling

OpenShift allows one to scale up/down the number of pods for each part of an application as needed.  This can be accomplished via changing our *replicaset/deployment* definition (declarative), by the command line (imperative), or via the web console (imperative). In our *deployment* definition (part of our `ostoy-fe-deployment.yaml`) we stated that we only want one pod for our microservice to start with. This means that the Kubernetes Replication Controller will always strive to keep one pod alive. We can also define pod autoscaling using the [Horizontal Pod Autoscaler](https://docs.openshift.com/container-platform/latest/nodes/pods/nodes-pods-autoscaling.html) (HPA) based on load to expand past what we defined. We will do this in a later section of this lab.

If we look at the tile on the left we should see one box randomly changing colors.  This box displays the randomly generated color sent to the frontend by our microservice along with the pod name that sent it. Since we see only one box that means there is only one microservice pod.  We will now scale up our microservice pods and will see the number of boxes change.

1. To confirm that we only have one pod running for our microservice, run the following command, or use the web console.

   ```
   oc get pods
   ```

1. The output will be similar to the following:

   ```
   $ oc get pods
   NAME                                   READY     STATUS    RESTARTS   AGE
   ostoy-frontend-679cb85695-5cn7x       1/1       Running   0          1h
   ostoy-microservice-86b4c6f559-p594d   1/1       Running   0          1h
   ```

1. Let's change our microservice definition yaml to reflect that we want 3 pods instead of the one we see. Download the [ostoy-microservice-deployment.yaml](https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/ostoy-microservice-deployment.yaml) file using the following command:

   ```
   curl -O https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/ostoy-microservice-deployment.yaml
   ```

1. Open the file using your favorite editor. Ex: `vi ostoy-microservice-deployment.yaml`.

1. Enter insert mode by pressing **i**.

1. Find the line that states `replicas: 1` and change that to `replicas: 3`. Then save and quit using the command **:wq** .

1. It will look like this

   ```
   spec:
      selector:
         matchLabels:
         app: ostoy-microservice
      replicas: 3
   ```

1. Assuming you are still logged in via the CLI, execute the following command:

   ```
   oc apply -f ostoy-microservice-deployment.yaml
   ```

1. Confirm that there are now 3 pods via the CLI (`oc get pods`) or the web console (*Workloads > Deployments > ostoy-microservice*).

1. See this visually by visiting the OSToy app and seeing how many boxes you now see.  It should be three.

   ![UI Scale](../media/managedlab/22-ostoy-colorspods.png)

1. Now we will scale the pods down using the command line.  Execute the following command from the CLI:

   ```
   oc scale deployment ostoy-microservice --replicas=2
   ```

1. Confirm that there are indeed 2 pods, via the CLI (`oc get pods`) or the web console.

1. See this visually by visiting the OSToy App and seeing how many boxes you now see.  It should be two.

1. Lastly, let's use the web console to scale back down to one pod.  Make sure you are in the project you created for this app (i.e., "ostoy"), in the left menu click *Workloads > Deployments > ostoy-microservice*.  On the left you will see a blue circle with the number 2 in the middle. Click on the down arrow to the right of that to scale the number of pods down to 1.

   ![UI Scale](../media/managedlab/21-ostoy-uiscale.png)

1. See this visually by visiting the OSToy app and seeing how many boxes you now see.  It should be one.  You can also confirm this via the CLI or the web console.

## Task 8: Pod Autoscaling

In this section we will explore how the [Horizontal Pod Autoscaler](https://docs.openshift.com/container-platform/latest/nodes/pods/nodes-pods-autoscaling.html) (HPA) can be used and works within Kubernetes/OpenShift.

As defined in the documentation:
> [...] you can use a horizontal pod autoscaler (HPA) to specify how OpenShift Container Platform should automatically increase or decrease the scale of a replication controller or deployment configuration, based on metrics collected from the pods that belong to that replication controller or deployment configuration.

In more simple words, "if there is a lot of work, make more pods".

We will create a HPA and then use OSToy to generate CPU intensive workloads.  We will then observe how the HPA will scale up the number of pods in order to handle the increased workloads.  

### 1. Create the Horizontal Pod Autoscaler

1. Run the following command to create the HPA. This will create an HPA that maintains between 1 and 10 replicas of the pods controlled by the *ostoy-microservice* deployment created. Roughly speaking, the HPA will increase and decrease the number of replicas (via the deployment) to maintain an average CPU utilization across all pods of 80% (since each pod requests 50 millicores, this means average CPU usage of 40 millicores).

   ```
   oc autoscale deployment/ostoy-microservice --cpu-percent=80 --min=1 --max=10
   ```

### 2. View the current number of pods

1. In the OSToy app in the left menu, click on "Autoscaling" to access this portion of the workshop.  

   ![HPA Menu](../media/managedlab/lab1-task8-2.png)

1. As was in the networking section you will see the total number of pods available for the microservice by counting the number of colored boxes.  In this case we have only one.  This can be verified through the web console or from the CLI.

   ![HPA Menu](../media/managedlab/lab1-task8-3.png)

1. You can use the following command to see the running microservice pods only:

   ```
   oc get pods --field-selector=status.phase=Running | grep microservice
   ```

### 3. Increase the load

1. Since we now only have one pod, let's increase the workload that the pod needs to perform. Click the link in the center of the card that says **"increase the load"**.  **Please click only *ONCE*!**

   ![HPA Menu](../media/managedlab/increase-load.png)

1. This will generate some CPU intensive calculations.  (If you are curious about what it is doing you can click [here](https://github.com/openshift-cs/ostoy/blob/master/microservice/app.js#L32)).

   > **Note:** The page may become slightly unresponsive.  This is normal; so be patient while the new pods spin up.

### 4. See the pods scale up

1. After about a minute the new pods will show up on the page (represented by the colored rectangles). Confirm that the pods did indeed scale up through the OpenShift Web Console or the CLI (you can use the command above).

   > **Note:** The page may still lag a bit which is normal.

1. You can see in this case it scaled up 2 more microservice pods (for a total of 3), using the command 

   ```
   oc get pods
   ```
1. The output will be similar to the following:

   ```
   $ oc get pods 
   NAME                                READY   STATUS    RESTARTS       AGE
   ostoy-frontend-64c8668694-7pq6k     1/1     Running   3 (105m ago)   23h
   ostoy-microservice-cf8bfb4c-bkwx8   1/1     Running   0              104s
   ostoy-microservice-cf8bfb4c-j24f9   1/1     Running   0              23h
   ostoy-microservice-cf8bfb4c-xls2t   1/1     Running   0              104s
   ```

### 5. Review resources in included observability

1. In the OpenShift web console left menu, click on *Observe > Dashboards*

1. In the dashboard, select *Kubernetes / Compute Resources / Namespace (Pods)* and our namespace *ostoy*.

   ![select_metrics](../media/managedlab/34-hpametrics.png)

1. Wait a few minutes and colorful graphs will appear showing resource usage across CPU and memory. The top graph will show recent CPU consumption per pod and the lower graph will indicate memory usage. Looking at this graph you can see how things developed. As soon as the load started to increase (A), two new pods started to spin up (B, C). The thickness of each graph is its CPU consumption indicating which pods handled more load. We also see that the load decreased (D), after which, the pods were spun back down.

   ![select_metrics](../media/managedlab/35-metrics.png)

1. At this point feel free to go back to the [logging section](#lab2-logging) to view this data through Container Insights for Azure Arc-enabled Kubernetes clusters.

## Task 9: Managing Worker Nodes

There may be times when you need to change aspects of your worker nodes. Things like scaling, changing the type, adding labels or taints to name a few. Most of these things are done through the use of machine sets. A machine is a unit that describes the host for a node and a machine set is a group of machines. Think of a machine set as a “template” for the kinds of machines that make up the worker nodes of your cluster. Similar to how a replicaset is to pods. A machine set allows users to manage many machines as a single entity though it is contained to a specific availability zone. If you’d like to learn more see [Overview of machine management](https://docs.openshift.com/container-platform/4.16/machine_management/index.html)

### Scaling worker nodes

### View the machine sets that are in the cluster

1. Let’s see which machine sets we have in our cluster. If you are following this lab, you should only have three so far (one for each availability zone). From the terminal run:

   ```
   oc get machinesets -n openshift-machine-api
   ```

1. You will see a response like:

   ```
   $ oc get machinesets -n openshift-machine-api
   NAME                           DESIRED   CURRENT   READY   AVAILABLE   AGE
   ok0620-rq5tl-worker-westus21   1         1         1       1           72m
   ok0620-rq5tl-worker-westus22   1         1         1       1           72m
   ok0620-rq5tl-worker-westus23   1         1         1       1           72m
   ```

1. This is telling us that there is a machine set defined for each availability zone in westus2 and that each has one machine.

### View the machines that are in the cluster

1. Let’s see which machines (nodes) we have in our cluster. From the terminal run:

   ```
   oc get machine -n openshift-machine-api
   ```

1. You will see a response like:

   ```
   $ oc get machine -n openshift-machine-api
   NAME                                 PHASE     TYPE              REGION    ZONE   AGE
   ok0620-rq5tl-master-0                Running   Standard_D8s_v3   westus2   1      73m
   ok0620-rq5tl-master-1                Running   Standard_D8s_v3   westus2   2      73m
   ok0620-rq5tl-master-2                Running   Standard_D8s_v3   westus2   3      73m
   ok0620-rq5tl-worker-westus21-n6lcs   Running   Standard_D4s_v3   westus2   1      73m
   ok0620-rq5tl-worker-westus22-ggcmv   Running   Standard_D4s_v3   westus2   2      73m
   ok0620-rq5tl-worker-westus23-hzggb   Running   Standard_D4s_v3   westus2   3      73m
   ```

1. As you can see we have 3 master nodes, 3 worker nodes, the types of nodes, and which region/zone they are in.

### Scale the number of nodes up via the CLI

1. Now that we know that we have 3 worker nodes, let’s scale the cluster up to have 4 worker nodes. We can accomplish this through the CLI or through the OpenShift Web Console. We’ll explore both.

1. From the terminal run the following to imperatively scale up a machine set to 2 worker nodes for a total of 4. Remember that each machine set is tied to an availability zone so with 3 machine sets with 1 machine each, in order to get to a TOTAL of 4 nodes we need to select one of the machine sets to scale up to 2 machines.

   ```
   oc scale --replicas=2 machineset <machineset> -n openshift-machine-api
   ```
1. The output will be similar to the following:

   ```$ oc scale --replicas=2 machineset ok0620-rq5tl-worker-westus23 -n openshift-machine-api
   machineset.machine.openshift.io/ok0620-rq5tl-worker-westus23 scaled
   ```

1. View the machine set by running the following command in terminal:

   ```
   oc get machinesets -n openshift-machine-api
   ```

1. You will now see that the desired number of machines in the machine set we scaled is “2”.

   ```
   $ oc get machinesets -n openshift-machine-api
   NAME                           DESIRED   CURRENT   READY   AVAILABLE   AGE
   ok0620-rq5tl-worker-westus21   1         1         1       1           73m
   ok0620-rq5tl-worker-westus22   1         1         1       1           73m
   ok0620-rq5tl-worker-westus23   2         2         1       1           73m
   ```

1. To check the machines in the clusters, run the following command in terminal

   ```
   oc get machine -n openshift-machine-api
   ```

1. You will see that one is in the “Provisioned” phase (and in the zone of the machineset we scaled) and will shortly be in “running” phase.

   ```
   $ oc get machine -n openshift-machine-api
   NAME                                 PHASE         TYPE              REGION    ZONE   AGE
   ok0620-rq5tl-master-0                Running       Standard_D8s_v3   westus2   1      74m
   ok0620-rq5tl-master-1                Running       Standard_D8s_v3   westus2   2      74m
   ok0620-rq5tl-master-2                Running       Standard_D8s_v3   westus2   3      74m
   ok0620-rq5tl-worker-westus21-n6lcs   Running       Standard_D4s_v3   westus2   1      74m
   ok0620-rq5tl-worker-westus22-ggcmv   Running       Standard_D4s_v3   westus2   2      74m
   ok0620-rq5tl-worker-westus23-5fhm5   Provisioned   Standard_D4s_v3   westus2   3      54s
   ok0620-rq5tl-worker-westus23-hzggb   Running       Standard_D4s_v3   westus2   3      74m
   ```

### Scale the number of nodes down via the Web Console

1. Now let’s scale the cluster back down to a total of 3 worker nodes, but this time, from the web console. (If you need the URL or credentials in order to access it please go back to the relevant portion of Lab 1)

1. Access your OpenShift web console from the relevant URL. If you need to find the URL you can run:

`az aro show --name <CLUSTER-NAME> --resource-group <RESOURCEGROUP> --query "consoleProfile.url" -o tsv`

1. Expand “Compute” in the left menu and then click on “MachineSets”

   ![](../media/managedlab/3.9-scale-nodes-1.png)

1. In the main pane you will see the same information about the machine sets from the command line. Now click on the “three dots” at the end of the line for the machine set that you scaled up to “2”. Select “Edit machine count” and decrease it to “1”. Click save.

   ![](../media/managedlab/3.9-scale-nodes-2.png)

### Cluster Autoscaling

The cluster autoscaler adjusts the size of an OpenShift Container Platform cluster to meet its current deployment needs. The cluster autoscaler increases the size of the cluster when there are pods that fail to schedule on any of the current worker nodes due to insufficient resources or when another node is necessary to meet deployment needs. The cluster autoscaler does not increase the cluster resources beyond the limits that you specify. To learn more visit the documentation for [cluster autoscaling](https://docs.openshift.com/container-platform/4.16/machine_management/applying-autoscaling.html).

A ClusterAutoscaler must have at least 1 machine autoscaler in order for the cluster autoscaler to scale the machines. The cluster autoscaler uses the annotations on machine sets that the machine autoscaler sets to determine the resources that it can scale. If you define a cluster autoscaler without also defining machine autoscalers, the cluster autoscaler will never scale your cluster.

### Create a Machine Autoscaler

This can be accomplished via the Web Console or through the CLI with a YAML file for the custom resource definition. We’ll use the latter.

1. Download the sample [MachineAutoscaler resource definition](https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/machine-autoscaler.yaml) and open it in your favorite editor using the following commands: 

   ```
   curl -O https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/machine-autoscaler.yaml
   vi machine-autoscaler.yaml
   ```

1. Enter insert mode by pressing **i**. For **metadata.name** give this machine autoscaler a name. Technically, this can be anything you want. But to make it easier to identify which machine set this machine autoscaler affects, specify or include the name of the machine set to scale. The machine set name takes the following form: <clusterid>-<machineset>-<region-az>.

1. For **spec.ScaleTargetRef.name** enter the name of the exact MachineSet you want this to apply to. Below is an example of a completed file.

   ```
   apiVersion: "autoscaling.openshift.io/v1beta1"
   kind: "MachineAutoscaler"
   metadata:
   name: "ok0620-rq5tl-worker-westus21-autoscaler"
   namespace: "openshift-machine-api"
   spec:
   minReplicas: 1
   maxReplicas: 7
   scaleTargetRef:
      apiVersion: machine.openshift.io/v1beta1
      kind: MachineSet
      name: ok0620-rq5tl-worker-westus21
   ```

1. Save and quit file using the command **:wq** .

1. Then create the resource in the cluster. Assuming you kept the same filename:

   ```
   oc create -f machine-autoscaler.yaml
   ```
1. You should see similar response:
   ```
   $ oc create -f machine-autoscaler.yaml
   machineautoscaler.autoscaling.openshift.io/ok0620-rq5tl-worker-westus21-mautoscaler created
   ```

1. You can also confirm this by checking the web console under “MachineAutoscalers” or by running:

   ```
   oc get machineautoscaler -n openshift-machine-api
   ```

1. You should see similar response:
   ```
   $ oc get machineautoscaler -n openshift-machine-api
   NAME                           REF KIND     REF NAME                      MIN   MAX   AGE
   ok0620-rq5tl-worker-westus21   MachineSet   ok0620-rq5tl-worker-westus2   1     7     40s
   ```

### Create the Cluster Autoscaler

This is the sample [ClusterAutoscaler resource definition](https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/cluster-autoscaler.yaml) for this lab.

See the [documentation](https://docs.openshift.com/container-platform/4.16/machine_management/applying-autoscaling.html#cluster-autoscaler-cr_applying-autoscaling) for a detailed explanation of each parameter. You shouldn’t need to edit this file.

1. Create the resource in the cluster using the following command:

   ```
   oc create -f https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/cluster-autoscaler.yaml
   ```

1. You should see similar response:
   ```
   $ oc create -f https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/cluster-autoscaler.yaml
   clusterautoscaler.autoscaling.openshift.io/default created
   ```

### Test the Cluster Autoscaler

Now we will test this out. Create a new project where we will define a job with a load that this cluster cannot handle. This should force the cluster to autoscale to handle the load.

1. Create a new project called “autoscale-ex” using the following command:

   ```
   oc new-project autoscale-ex
   ```

1. Create the job by running the below command:

   ```
   oc create -f https://raw.githubusercontent.com/openshift/training/master/assets/job-work-queue.yaml
   ```

1. After a few seconds, run the following to see what pods have been created by using the following command:

   ```
   oc get pods
   ```
1. You should see similar response:
   ```
   $ oc get pods
   NAME                     READY   STATUS              RESTARTS   AGE
   work-queue-28n9m-29qgj   1/1     Running             0          53s
   work-queue-28n9m-2c9rm   0/1     Pending             0          53s
   work-queue-28n9m-57vnc   0/1     Pending             0          53s
   work-queue-28n9m-5gz7t   0/1     Pending             0          53s
   work-queue-28n9m-5h4jv   0/1     Pending             0          53s
   work-queue-28n9m-6jz7v   0/1     Pending             0          53s
   work-queue-28n9m-6ptgh   0/1     Pending             0          53s
   work-queue-28n9m-78rr9   1/1     Running             0          53s
   work-queue-28n9m-898wn   0/1     ContainerCreating   0          53s
   work-queue-28n9m-8wpbt   0/1     Pending             0          53s
   work-queue-28n9m-9nm78   1/1     Running             0          53s
   work-queue-28n9m-9ntxc   1/1     Running             0          53s
   [...]
   ```

1. We see a lot of pods in a pending state. This should trigger the cluster autoscaler to create more machines using the MachineAutoscaler we created. If we check on the MachineSets using the command:

   ```
   oc get machinesets -n openshift-machine-api
   ```

1. You should see similar response:
   ```
   $ oc get machinesets -n openshift-machine-api
   NAME                           DESIRED   CURRENT   READY   AVAILABLE   AGE
   ok0620-rq5tl-worker-westus21   5         5         1       1           7h17m
   ok0620-rq5tl-worker-westus22   1         1         1       1           7h17m
   ok0620-rq5tl-worker-westus23   1         1         1       1           7h17m
   ```

1. We see that the cluster autoscaler has already scaled the machine set up to 5 in our example. Though it is still waiting for those machines to be ready.

1. If we check on the machines we should see that 4 are in a “Provisioned” state (there was 1 already existing from before for a total of 5 in this machine set).

   ```
   $ oc get machines -n openshift-machine-api
   NAME                                 PHASE         TYPE              REGION    ZONE   AGE
   ok0620-rq5tl-master-0                Running       Standard_D8s_v3   westus2   1      7h18m
   ok0620-rq5tl-master-1                Running       Standard_D8s_v3   westus2   2      7h18m
   ok0620-rq5tl-master-2                Running       Standard_D8s_v3   westus2   3      7h18m
   ok0620-rq5tl-worker-westus21-7hqgz   Provisioned   Standard_D4s_v3   westus2   1      72s
   ok0620-rq5tl-worker-westus21-7j22r   Provisioned   Standard_D4s_v3   westus2   1      73s
   ok0620-rq5tl-worker-westus21-7n7nf   Provisioned   Standard_D4s_v3   westus2   1      72s
   ok0620-rq5tl-worker-westus21-8m94b   Provisioned   Standard_D4s_v3   westus2   1      73s
   ok0620-rq5tl-worker-westus21-qnlfl   Running       Standard_D4s_v3   westus2   1      13m
   ok0620-rq5tl-worker-westus22-9dtk5   Running       Standard_D4s_v3   westus2   2      22m
   ok0620-rq5tl-worker-westus23-hzggb   Running       Standard_D4s_v3   westus2   3      7h15m
   ```

1. After a few minutes we should see all 5 are provisioned.

   ```
   $ oc get machinesets -n openshift-machine-api
   NAME                           DESIRED   CURRENT   READY   AVAILABLE   AGE
   ok0620-rq5tl-worker-westus21   5         5         5       5           7h23m
   ok0620-rq5tl-worker-westus22   1         1         1       1           7h23m
   ok0620-rq5tl-worker-westus23   1         1         1       1           7h23m
   ```

1. If we now wait a few more minutes for the pods to complete, we should see the cluster autoscaler begin scale down the machine set and thus delete machines.

   ```
   $ oc get machinesets -n openshift-machine-api
   NAME                           DESIRED   CURRENT   READY   AVAILABLE   AGE
   ok0620-rq5tl-worker-westus21   4         4         4       4           7h27m
   ok0620-rq5tl-worker-westus22   1         1         1       1           7h27m
   ok0620-rq5tl-worker-westus23   1         1         1       1           7h27m
   ```

   ```
   $ oc get machines -n openshift-machine-api
   NAME                                 PHASE      TYPE              REGION    ZONE   AGE
   ok0620-rq5tl-master-0                Running    Standard_D8s_v3   westus2   1      7h28m
   ok0620-rq5tl-master-1                Running    Standard_D8s_v3   westus2   2      7h28m
   ok0620-rq5tl-master-2                Running    Standard_D8s_v3   westus2   3      7h28m
   ok0620-rq5tl-worker-westus21-7hqgz   Running    Standard_D4s_v3   westus2   1      10m
   ok0620-rq5tl-worker-westus21-7j22r   Running    Standard_D4s_v3   westus2   1      10m
   ok0620-rq5tl-worker-westus21-8m94b   Deleting   Standard_D4s_v3   westus2   1      10m
   ok0620-rq5tl-worker-westus21-qnlfl   Running    Standard_D4s_v3   westus2   1      22m
   ok0620-rq5tl-worker-westus22-9dtk5   Running    Standard_D4s_v3   westus2   2      32m
   ok0620-rq5tl-worker-westus23-hzggb   Running    Standard_D4s_v3   westus2   3      7h24m
   ```

### Adding node labels

To add a node label it is recommended to set the label in the machine set. While you can directly add a label the node, this is not recommended since nodes could be overwritten and then the label would disappear. Once the machine set is modified to contain the desired label any new machines created from that set would have the newly added labels. This means that existing machines (nodes) will not get the label. Therefore, to make sure all nodes have the label, you should scale the machine set down to zero and then scale the machine set back up.

##### Using the web console

1. Select “MachineSets” from the left menu. You will see the list of machinesets.

   ![](../media/managedlab/3.9-machine-sets-3.png)

1. We’ll select the first one “ok0620-rq5tl-worker-westus21”

1. Click on the second tab “YAML”

1. Click into the YAML and under spec.template.spec.metadata add “labels:” then under that add a key:value pair for the label you want. In our example we can add a label “tier: frontend”. Click Save.

   ![](../media/managedlab/3.9-machine-sets-frontend-4.png)

1. The already existing machine won’t get this label but any new machines will. So to ensure that all machines get the label, we will scale down this machine set to zero, then once completed we will scale it back up as we did earlier.

1. Click on the node that was just created.

1. You can see that the label is now there.

   ![](../media/managedlab/3.9-nodes-frontend-5.png)

## Task 10: Azure Service Operator - Blob Store

### Integrating with Azure services

So far, our OSToy application has functioned independently without relying on any external services. While this may be nice for a workshop environment, it’s not exactly representative of real-world applications. Many applications require external services like databases, object stores, or messaging services.

In this section, we will learn how to integrate our OSToy application with other Azure services, specifically Azure Blob Storage and Key Vault. By the end of this section, our application will be able to securely create and read objects from Blob Storage.

To achieve this, we will use the Azure Service Operator (ASO) to create the necessary services for our application directly from Kubernetes. We will also utilize Key Vault to securely store the connection secret required for accessing the Blob Storage container. We will create a Kubernetes secret to retrieve this secret from Key Vault, enabling our application to access the Blob Storage container using the secret.

To demonstrate this integration, we will use OSToy to create a basic text file and save it in Blob Storage. Finally, we will confirm that the file was successfully added and can be read from Blob Storage.

#### Azure Service Operator (ASO)

The [Azure Service Operator](https://azure.github.io/azure-service-operator/) (ASO) allows you to create and use Azure services directly from Kubernetes. You can deploy your applications, including any required Azure services directly within the Kubernetes framework using a familiar structure to declaratively define and create Azure services like Storage Blob or CosmosDB databases.

#### Key Vault

Azure Key Vault is a cloud-based service provided by Microsoft Azure that allows you to securely store and manage cryptographic keys, secrets, and certificates used by your applications and services.

##### Why should you use Key Vault to store secrets?

Using a secret store like Azure Key Vault allows you to take advantage of a number of benefits.

1. Scalability - Using a secret store service is already designed to scale to handle a large number of secrets over placing them directly in the cluster.
1. Centralization - You are able to keep all your organizations secrets in one location.
1. Security - Features like access control, monitoring, encryption and audit are already baked in.
1. Rotation - Decoupling the secret from your cluster makes it much easier to rotate secrets since you only have to update it in Key Vault and the Kubernetes secret in the cluster will reference that external secret store. This also allows for separation of duties as someone else can manage these resources.

#### Section overview

To provide a clearer understanding of the process, the procedure we will be following consists of three primary parts.

1. **Install the Azure Service Operator** - This allows you to create/delete Azure services (in our case, Blob Storage) through the use of a Kubernetes Custom Resource. Install the controller which will also create the required namespace and the service account and then create the required resources.
1. **Setup Key Vault** - Perform required prerequisites (ex: install CSI drivers), create a Key Vault instance, add the connection string.
1. **Application access** - Configuring the application to access the stored connection string in Key Vault and thus enable the application to access the Blob Storage location.

Below is an updated application diagram of what this will look like after completing this section.

![](../media/managedlab/3.10-arch-diagram.png)

**Access the cluster** - Login to the cluster using the oc CLI if you are not already logged in.

#### Setup

##### Define helper variables

Set helper environment variables to facilitate execution of the commands in this section. Replace <REGION> with the Azure region you are deploying into (ex: eastus or westus2).

Replace the **{Application Id}** for SERVICE_PRINCIPAL_CLIENT_ID and **{Secret Key}** for SERVICE_PRINCIPAL_CLIENT_SECRET with the actual values by navigating to the **Environment > Service Principal Details** tab in your lab environment.

![](../media/managedlab/spn-details.png)

```
export AZURE_SUBSCRIPTION_ID=<inject key="Subscription ID" enableCopy="false"/>
export AZ_TENANT_ID=<inject key="Tenant ID" enableCopy="false"/>
export PROJECT_NAME=ostoy-app01
export KEYVAULT_NAME=keyvault<inject key="Deployment ID" enableCopy="false"/>
export REGION=<inject key="Region" enableCopy="false"/>
export SERVICE_PRINCIPAL_CLIENT_ID={Application Id}
export SERVICE_PRINCIPAL_CLIENT_SECRET={Secret Key}
```

##### Install Helm

1. Install Helm if you don’t already have it by running the below commands in the Azure cloudshell. You can also check the [Official Helm site](https://helm.sh/docs/intro/install/) for other install options.

   ```
   curl -LO https://get.helm.sh/helm-v3.15.4-linux-amd64.tar.gz
   tar -zxvf helm-v3.15.4-linux-amd64.tar.gz
   ```

1. Verify the Helm version installed.

   ```
   helm version
   ```
   
##### Install the Azure Service Operator - Set up ASO

1. We first need to install Cert Manager. Run the following.

   ```
   oc apply -f https://github.com/jetstack/cert-manager/releases/download/v1.8.2/cert-manager.yaml
   ```

1. Confirm that the cert-manager pods have started successfully before continuing.

   ```
   oc get pods -n cert-manager
   ```

1. You will see a response like:

   ```
   NAME                                       READY   STATUS    RESTARTS   AGE
   cert-manager-677874db78-t6wgn              1/1     Running   0          1m
   cert-manager-cainjector-6c5bf7b759-l722b   1/1     Running   0          1m
   cert-manager-webhook-5685fdbc4b-rlbhz      1/1     Running   0          1m
   ```

1. We then need to add the latest Helm chart for the ASO.

   ```
   helm repo add aso2 https://raw.githubusercontent.com/Azure/azure-service-operator/main/v2/charts
   ```

1. Update the Helm repository.

   ```
   helm repo update
   ```

1. Install the ASO.

   ```
   helm upgrade --install aso2 aso2/azure-service-operator \
    --create-namespace \
    --namespace=azureserviceoperator-system \
    --set crdPattern='resources.azure.com/*;containerservice.azure.com/*;keyvault.azure.com/*;managedidentity.azure.com/*;eventhub.azure.com/*'
   ```

1. Ensure that the pods are running successfully. This could take about 2 minutes.

   ```
   oc get pods -n azureserviceoperator-system
   ```

1. You will see a response like:

   ```
   NAME                                                      READY   STATUS    RESTARTS   AGE
   azureserviceoperator-controller-manager-5b4bfc59df-lfpqf   2/2     Running   0          24s
   ```

### Create Storage Accounts and containers using Azure portal

Now we need to create a Storage Account for our Blob Storage, to use with OSToy.

1. First, create a new OpenShift project for our OSToy app (even if you already have one from earlier).

   ```
   oc new-project $PROJECT_NAME
   ```

1. Navigate to Azure portal and select **Resource groups**.

   ![](../media/managedlab/resource-groups.png)

1. On the **Create a resource group** tab, specify the following settings and click **Review + create** and **Create**.

   - Subscription: Select your default subscription
   - Resource group: **ostoy-app01-rg**
   - Region: **<inject key="Region" enableCopy="false"/>**
  
      ![](../media/managedlab/create-rg.png)

1. In the search bar, search for storage and select **Storage accounts** and **Create** a new storage account.

   ![](../media/managedlab/search-strg-acc.png)

1. On the **Create a storage account** tab, specify the following settings and click **Next**.

   - Resource group: **ostoy-app01-rg**
   - Storage account name: **ostoystorage<inject key="Deployment ID" enableCopy="false"/>**
   - Region: **<inject key="Region" enableCopy="false"/>**
   - Primary service: **Azure Blob Storage or Azure Data Lake Storage Gen 2**
   - Performance: **Standard**
   - Redundancy: **Locally-redundant storage (LRS)**
  
1. On the **Advanced** tab, enable the **Allow enabling anonymous access on individual containers** and click **Review + create** and then **Create**.

   ![](../media/managedlab/create-review-strg.png)

1. Once the storage account deployment succeeds, click **Go to resource**.

   ![](../media/managedlab/strg-go-to-resource.png)

1. In the newly created storage account, navigate to **Data storage > Containers** settings and click **+ Container** to create new storage containers.

   ![](../media/managedlab/strg-containers.png)

1. Specify the new container name as **ostoystorage<inject key="Deployment ID" enableCopy="false"/>service**, select **Blob (anonymous read access for blobs only)** from the dropdown menu and click **Create**.

   ![](../media/managedlab/strg-containers-blob.png)

1. Create another container by specifying the container name as **ostoy-app01-container**, select **Container (anonymous read access for containers and blobs)** from the dropdown menu and click **Create**.

   ![](../media/managedlab/strg-containers-container.png)

1. In the storage account, navigate to **Security + Networking > Shared access signature**, enable all the permissions and click **Generate SAS and connection string** to create a new connection string and copy the **Connection string** value in a notepad. You will need this connection string value in the next tasks.

   ![](../media/managedlab/strg-sas.png)

   ![](../media/managedlab/strg-conn-string.png)

1. The storage account is now set up for use with our application.

#### Install Kubernetes Secret Store CSI

In this part we will create a Key Vault location to store the connection string to our Storage account. Our application will use this to connect to the Blob Storage container we created, enabling it to display the contents, create new files, as well as display the contents of the files. We will mount this as a secret in a secure volume mount within our application. Our application will then read that to access the Blob storage.

1. To simplify the process for the workshop, a script is provided that will do the prerequisite work in order to use Key Vault stored secrets, run the below command in **CloudShell**. If you are curious, please feel free to read the script, otherwise just run it. This should take about 1-2 minutes to complete.

   ```
   curl https://raw.githubusercontent.com/microsoft/aroworkshop/master/resources/setup-csi.sh | bash
   ```
   Or, if you’d rather not live on the edge, feel free to download it first.

   >**NOTE:** This command might take more than expected time to execute completely. In this case, open cloudshell window in another browser/tab, login to the OpenShift console by running the **oc login** command and define the variables again under **Define helper variables** section of this task to proceed with the next steps.

   >**NOTE:** Instead, you could connect your cluster to Azure ARC and use the [KeyVault extension](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/tutorial-akv-secrets-provider)

1. In the search bar, search for key vaults and select **Key vaults** and **Create** a new key vault.

   ![](../media/managedlab/search-key-vaults.png)

1. On the **Create a key vault** tab, specify the following settings and click **Next**.

   - Resource group: **ostoy-app01-rg**
   - Key vault name: **keyvault<inject key="Deployment ID" enableCopy="false"/>**
   - Region: **<inject key="Region" enableCopy="false"/>**
   - Pricing tier: **Standard**
  
   ![](../media/managedlab/key-vault-basics.png)
  
1. On the **Access configuration** tab, select **Vault access policy** as Permission model and click **+ Create** under Access policies.

   ![](../media/managedlab/key-vault-access-policy.png)

1. On the **Create an access policy > Permissions** tab, select all the permissions for **Secret permissions** and click **Next**.

   ![](../media/managedlab/access-policy-permissions.png)

1. On the **Principal** tab, search for the service principal **https://odl_user_sp_<inject key="Deployment ID" enableCopy="false"/>**, select it and click **Next**.

   ![](../media/managedlab/access-policy-principal.png)

1. On the **Review + create** tab, review the access policy settings and click **Create**.

   ![](../media/managedlab/access-policy-create.png)

1. Back on Create a key vault page, click on **Review + create** > **Create**.

1. Now that the access policies are set up, create the key vault. Once the key vault deployment succeeds, click **Go to resource**.

   ![](../media/managedlab/key-vault-go-to-resource.png)

1. In the newly created key vault, navigate to **Objects > Secrets** settings and click **+ Generate/Import** to create a new secret.

   ![](../media/managedlab/key-vault-secrets.png)

1. On the **Create a secret** tab, specify the name as **connectionsecret**, paste the storage account connection string from the previous task and click **Create**.

   ![](../media/managedlab/key-vault-secrets-create.png)

1. Create a secret for Kubernetes to use to access the Key Vault. When this command is executed, the Service Principal’s credentials are stored in the **secrets-store-creds** Secret object, where it can be used by the Secret Store CSI driver to authenticate with Azure Key Vault and retrieve secrets when needed.

   ```
   oc create secret generic secrets-store-creds -n $PROJECT_NAME --from-literal clientid=$SERVICE_PRINCIPAL_CLIENT_ID --from-literal clientsecret=$SERVICE_PRINCIPAL_CLIENT_SECRET
   ```

1. Create a label for the secret. By default, the secret store provider has filtered watch enabled on secrets. You can allow it to find the secret in the default configuration by adding this label to the secret.

   ```
   oc -n $PROJECT_NAME label secret secrets-store-creds secrets-store.csi.k8s.io/used=true
   ```

1. Create the Secret Provider Class to give access to this secret. To learn more about the fields in this class see [Secret Provider Class](https://learn.microsoft.com/en-us/azure/aks/hybrid/secrets-store-csi-driver#create-and-apply-your-own-secretproviderclass-object) object.

   ```
   cat <<EOF | oc apply -f -
   apiVersion: secrets-store.csi.x-k8s.io/v1
   kind: SecretProviderClass
   metadata:
     name: azure-kvname
     namespace: $PROJECT_NAME
   spec:
     provider: azure
     parameters:
       usePodIdentity: "false"
       useVMManagedIdentity: "false"
       userAssignedIdentityID: ""
       keyvaultName: "${KEYVAULT_NAME}"
       objects: |
         array:
           - |
             objectName: connectionsecret
             objectType: secret
             objectVersion: ""
       tenantId: "${AZ_TENANT_ID}"
   EOF
   ```

<validation step="58614025-40b6-49fa-ac36-a00931c3dae0" />

#### Create a custom Security Context Constraint (SCC)

SCCs are outside the scope of this workshop. Though, in short, OpenShift SCCs are a mechanism for controlling the actions and resources that a pod or container can access in an OpenShift cluster. SCCs can be used to enforce security policies at the pod or container level, which helps to improve the overall security of an OpenShift cluster. For more details please see [Managing security context constraints](https://docs.openshift.com/container-platform/4.16/authentication/managing-security-context-constraints.html).

1. Create a new SCC that will allow our OSToy app to use the Secrets Store Provider CSI driver. The SCC that is used by default, **restricted**, does not allow it. So in this custom SCC we are explicitly allowing access to CSI. If you are curious feel free to view the file first, the last line in specific.

   ```
   oc apply -f https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/ostoyscc.yaml
   ```

1. Create a Service Account for the application.

   ```
   oc create sa ostoy-sa -n $PROJECT_NAME
   ```

1. Grant permissions to the Service Account using the custom SCC we just created.

   ```
   oc adm policy add-scc-to-user ostoyscc system:serviceaccount:${PROJECT_NAME}:ostoy-sa
   ```

#### Deploy the OSToy application

1. Deploy the application. First deploy the microservice.

   ```
   oc apply -n $PROJECT_NAME -f https://raw.githubusercontent.com/microsoft/aroworkshop/master/yaml/ostoy-microservice-deployment.yaml
   ```

1. Run the following to deploy the frontend. This will automatically remove the comment symbols for the new lines that we need in order to use the secret.

   ```
   curl https://raw.githubusercontent.com/CloudLabsAI-Azure/aroworkshop/master/yaml/ostoy-frontend-deployment.yaml | oc apply -n $PROJECT_NAME -f -
   ```

   >**Note:** To view the YAML file with comments https://raw.githubusercontent.com/openshift-cs/rosaworkshop/master/rosa-workshop/ostoy/yaml/ostoy-frontend-deployment.yaml

#### See the storage contents through OSToy - READ-ONLY

After about a minute we can use our app to see the contents of our Blob storage container.

1. Get the route for the newly deployed application.

   ```
   oc get route ostoy-route -n $PROJECT_NAME -o jsonpath='{.spec.host}{"\n"}'
   ```

1. Open a new browser tab and enter the route from above. Ensure that it is using **http://** and not **https://**.

1. A new menu item will appear. Click on “ASO - Blob Storage” in the left menu in OSToy.

1. You will see a page that lists the contents of the bucket, which at this point should be empty.

   ![](../media/managedlab/3.10-aso-blob-strg.png)

1. Move on to the next step to add some files.

#### Create files in your Azure Blob Storage Container - READ-ONLY

For this step we will use OStoy to create a file and upload it to the Blob Storage Container. While Blob Storage can accept any kind of file, for this workshop we’ll use text files so that the contents can easily be rendered in the browser.

1. Click on “ASO - Blob Storage” in the left menu in OSToy.

1. Scroll down to the section underneath the “Existing files” section, titled “Upload a text file to Blob Storage”.

1. Enter a file name for your file.

1. Enter some content for your file.

1. Click “Create file”.

   ![](../media/managedlab/3.10-create-file.png)

1. Scroll up to the top section for existing files and you should see your file that you just created there.

1. Click on the file name to view the file.

   ![](../media/managedlab/3.10-create-file-name.png)

1. Now to confirm that this is not just some smoke and mirrors, let’s confirm directly via the CLI. Run the following to list the contents of our bucket.

   ```
   az storage blob list --account-name ostoystorage${MY_UUID} --connection-string $CONNECTION_STRING -c ${PROJECT_NAME}-container --query "[].name" -o tsv
   ```

   We should see our file(s) returned.

## Reference Links

- [Access modes](https://docs.openshift.com/container-platform/latest/storage/understanding-persistent-storage.html#pv-access-modes_understanding-persistent-storage)
- [ConfigMaps](https://docs.openshift.com/container-platform/latest/nodes/pods/nodes-pods-configmaps.html)
- [Secrets](https://docs.openshift.com/container-platform/latest/cicd/builds/creating-build-inputs.html#builds-input-secrets-configmaps_creating-build-inputs)
- [Environment Variables](https://docs.openshift.com/container-platform/3.11/dev_guide/environment_variables.html)
- [OVS-networkpolicy](https://docs.openshift.com/container-platform/latest/networking/network_policy/about-network-policy.html#nw-networkpolicy-about_about-network-policy)
- [Horizontal Pod Autoscaler](https://docs.openshift.com/container-platform/latest/nodes/pods/nodes-pods-autoscaling.html)
- [Overview of machine management](https://docs.openshift.com/container-platform/4.16/machine_management/index.html)
- [Azure Service Operator](https://azure.github.io/azure-service-operator/)

## Summary

You will cover an application overview, deployment, logging, metrics, health checks, persistent storage, configuration, networking, scaling, pod autoscaling, managing worker nodes, and using Azure Service Operator for Blob Store.

### You have successfully completed the lab








