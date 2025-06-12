# Prerequisites

### Estimated Duration: 15 minutes

## Prerequisites Objectives

You will be able to complete the following tasks:

- Task 1: Setup Azure Cloud Shell
- Task 2: Install OpenShift CLI (oc)
- Task 3: GitHub Account

## Task 1: Setup Azure Cloud Shell

Azure Cloud Shell provides a browser-based command-line environment with pre-installed Azure tools. This lab uses Cloud Shell to run OpenShift CLI (oc) commands without needing local installations.

You can use the Azure Cloud Shell accessible at <https://shell.azure.com> once you login with an Azure subscription.

Head over to <https://shell.azure.com> and sign in with your Azure Subscription details.

1. Select **Bash** as your shell.

   ![](../media/cloudshell/select-bash.png)

1. Select **Mount storage account**, your default **Subscription** and click **Apply**.

   ![](../media/Redhat-image1.png)

1. Select **I want to create a storage account** and click **Next**.

   ![](../media/cloudshell/select-create-strg.png)

1. Specify then following values and click **Create (6)** to create a new storage account.

   - Subscription: **Select your default subscription (1)**
   - Resource group: **openshift (2)**
   - Region: **<inject key="Region" enableCopy="false"/> (3)**
   - Storage account name: **strg<inject key="Deployment ID" enableCopy="false"/> (4)**
   - File share: **none (5)**

   ![](../media/Redhat-image2.png)

1. You should now have access to the Azure Cloud Shell.

   ![](../media/Redhat-image3.png)

<validation step="e4da372d-001a-4680-ba58-23f917916623" />

## Task 2: Install OpenShift CLI (oc)

The OpenShift CLI (oc) must be installed in the Azure Cloud Shell environment to interact with the Azure Red Hat OpenShift (ARO) cluster, manage resources, and deploy workloads through command-line operations.

You'll need to download the **latest OpenShift CLI (oc)** client tools for OpenShift 4. You can follow the steps below on the Azure Cloud Shell.

Please run following commands on Azure Cloud Shell to download and setup the OpenShift client.

```sh
cd ~
curl https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz > openshift-client-linux.tar.gz

mkdir openshift

tar -zxvf openshift-client-linux.tar.gz -C openshift

echo 'export PATH=$PATH:~/openshift' >> ~/.bashrc && source ~/.bashrc

```

The OpenShift CLI (oc) is now installed.

In case you want to work from your own operating system, here are the links to the different versions of CLI:

- https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-windows.zip
- https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
- https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-mac.tar.gz

## Task 3: GitHub Account

A GitHub account is required to access and clone the Rating API web application repository. This app will be deployed on the ARO cluster using MongoDB pods and managed via the ARO web console.

You'll need a personal GitHub account. If not signed-up already then, you can sign up for free [here](https://github.com/join).

### You are all setup with the prerequisites!

-----

# READ-ONLY

## Basic concepts

### Source-To-Image (S2I)

Source-to-Image (S2I) is a toolkit and workflow for building reproducible container images from source code. S2I produces ready-to-run images by injecting source code into a container image and letting the container prepare that source code for execution. By creating self-assembling builder images, you can version and control your build environments exactly like you use container images to version your runtime environments.

#### How it works

For a dynamic language like Ruby, the build-time and run-time environments are typically the same. Starting with a builder image that describes this environment - with Ruby, Bundler, Rake, Apache, GCC, and other packages needed to set up and run a Ruby application installed - source-to-image performs the following steps:

1. Start a container from the builder image with the application source injected into a known directory

1. The container process transforms that source code into the appropriate runnable setup - in this case, by installing dependencies with Bundler and moving the source code into a directory where Apache has been preconfigured to look for the Ruby config.ru file.

1. Commit the new container and set the image entrypoint to be a script (provided by the builder image) that will start Apache to host the Ruby application.

For compiled languages like C, C++, Go, or Java, the dependencies necessary for compilation might dramatically outweigh the size of the actual runtime artifacts. To keep runtime images slim, S2I enables a multiple-step build processes, where a binary artifact such as an executable or Java WAR file is created in the first builder image, extracted, and injected into a second runtime image that simply places the executable in the correct location for execution.

For example, to create a reproducible build pipeline for Tomcat (the popular Java webserver) and Maven:

1. Create a builder image containing OpenJDK and Tomcat that expects to have a WAR file injected

1. Create a second image that layers on top of the first image Maven and any other standard dependencies, and expects to have a Maven project injected

1. Invoke source-to-image using the Java application source and the Maven image to create the desired application WAR

1. Invoke source-to-image a second time using the WAR file from the previous step and the initial Tomcat image to create the runtime image

By placing our build logic inside of images, and by combining the images into multiple steps, we can keep our runtime environment close to our build environment (same JDK, same Tomcat JARs) without requiring build tools to be deployed to production.

#### Goals and benefits

##### Reproducibility

Allow build environments to be tightly versioned by encapsulating them within a container image and defining a simple interface (injected source code) for callers. Reproducible builds are a key requirement to enabling security updates and continuous integration in containerized infrastructure, and builder images help ensure repeatability as well as the ability to swap runtimes.

##### Flexibility

Any existing build system that can run on Linux can be run inside of a container, and each individual builder can also be part of a larger pipeline. In addition, the scripts that process the application source code can be injected into the builder image, allowing authors to adapt existing images to enable source handling.

##### Speed

Instead of building multiple layers in a single Dockerfile, S2I encourages authors to represent an application in a single image layer. This saves time during creation and deployment, and allows for better control over the output of the final image.

##### Security

Dockerfiles are run without many of the normal operational controls of containers, usually running as root and having access to the container network. S2I can be used to control what permissions and privileges are available to the builder image since the build is launched in a single container. In concert with platforms like OpenShift, source-to-image can enable admins to tightly control what privileges developers have at build time.

### Routes

An OpenShift `Route` exposes a service at a host name, like www.example.com, so that external clients can reach it by name. When a `Route` object is created on OpenShift, it gets picked up by the built-in HAProxy load balancer in order to expose the requested service and make it externally available with the given configuration. You might be familiar with the Kubernetes `Ingress` object and might already be asking "what's the difference?". Red Hat created the concept of `Route` in order to fill this need and then contributed the design principles behind this to the community; which heavily influenced the `Ingress` design.  Though a `Route` does have some additional features as can be seen in the chart below.

![routes vs ingress](../media/managedlab/routes-vs-ingress.png)

> **NOTE:** DNS resolution for a host name is handled separately from routing; your administrator may have configured a cloud domain that will always correctly resolve to the router, or if using an unrelated host name you may need to modify its DNS records independently to resolve to the router.

Also of note is that an individual route can override some defaults by providing specific configuraitons in its annotations.  See here for more details: [https://docs.openshift.com/aro/4/networking/routes/route-configuration.html](https://docs.openshift.com/aro/4/networking/routes/route-configuration.html)

### ImageStreams

An ImageStream stores a mapping of tags to images, metadata overrides that are applied when images are tagged in a stream, and an optional reference to a Docker image repository on a registry.

#### What are the benefits

Using an ImageStream makes it easy to change a tag for a container image.  Otherwise to change a tag you need to download the whole image, change it locally, then push it all back. Also promoting applications by having to do that to change the tag and then update the deployment object entails many steps.  With ImageStreams you upload a container image once and then you manage it’s virtual tags internally in OpenShift.  In one project you may use the `dev` tag and only change reference to it internally, in prod you may use a `prod` tag and also manage it internally. You don't really have to deal with the registry!

You can also use ImageStreams in conjunction with DeploymentConfigs to set a trigger that will start a deployment as soon as a new image appears or a tag changes its reference.

See here for more details: [https://blog.openshift.com/image-streams-faq/](https://blog.openshift.com/image-streams-faq/) <br>
OpenShift Docs: [https://docs.openshift.com/aro/4/openshift_images/managing_images/managing-images-overview.html](https://docs.openshift.com/aro/4/openshift_images/managing_images/managing-images-overview.html)<br>
ImageStream and Builds: [https://cloudowski.com/articles/why-managing-container-images-on-openshift-is-better-than-on-kubernetes/](https://cloudowski.com/articles/why-managing-container-images-on-openshift-is-better-than-on-kubernetes/)

### Builds

A build is the process of transforming input parameters into a resulting object. Most often, the process is used to transform input parameters or source code into a runnable image. A BuildConfig object is the definition of the entire build process.

OpenShift Container Platform leverages Kubernetes by creating Docker-formatted containers from build images and pushing them to a container image registry.

Build objects share common characteristics: inputs for a build, the need to complete a build process, logging the build process, publishing resources from successful builds, and publishing the final status of the build. Builds take advantage of resource restrictions, specifying limitations on resources such as CPU usage, memory usage, and build or pod execution time.

See here for more details: [https://docs.openshift.com/aro/4/openshift_images/image-streams-manage.html](https://docs.openshift.com/aro/4/openshift_images/image-streams-manage.html)

## Differences Between Public and Private ARO Clusters

| Feature | Public ARO Cluster | Private ARO Cluster |
|---------|-------------------|---------------------|
| Ingress Controller | Uses default public ingress | Requires private ingress controller |
| Connectivity | Direct from Front Door to public endpoints | Requires Private Link Service |
| Default Security | Traffic flows over public internet to ARO | Traffic remains on Microsoft backbone |
| Implementation | Simpler, fewer components | More complex, more secure |
| Variable Requirements | PUBLIC_ROUTE_HOST | PRIVATE_LINK_SERVICE_ID |
| Route Configuration | Standard route | Route with private link backend |
| Network Configuration | No additional network setup | Requires proper VNet and subnet configuration |
| DNS Requirements | Direct DNS to public endpoints | DNS to Front Door endpoints only |

## Summary 

In this prerequisites section, you completed the following tasks:

- Setup Azure Cloud Shell
- Installed OpenShift CLI (oc)
- Setup your personal GitHub Account

### You have successfully completed setting up the prerequisites. Click on **Next>>** to proceed with the next lab.
