# Azure Red Hat OpenShift Workshop

[Azure Red Hat OpenShift](https://azure.microsoft.com/en-us/services/openshift/) is a fully managed Red Hat OpenShift service in Azure that is jointly engineered and supported by Microsoft and Red Hat. In this lab, you'll go through a set of tasks that will help you understand some of the concepts of deploying and securing container based applications on top of Azure Red Hat OpenShift.

You can use this guide as an OpenShift tutorial and as study material to help you get started to learn OpenShift.

Some of the things you’ll be going through:

- Creating a [project](https://docs.openshift.com/aro/4/applications/projects/working-with-projects.html) on the Azure Red Hat OpenShift Web Console
- Deploying a MongoDB container that uses Azure Disks for [persistent storage](https://docs.openshift.com/aro/4/storage/understanding-persistent-storage.html)
- Deploying a Node JS API and frontend app from Git Hub using [Source-To-Image (S2I)](https://docs.openshift.com/aro/4/openshift_images/create-images.html)
- Exposing the web application frontend using [Routes](https://docs.openshift.com/aro/4/networking/routes/route-configuration.html)
- Creating a [network policy](https://docs.openshift.com/aro/4/networking/network_policy/about-network-policy.html) to control communication between the different tiers in the application

You'll be doing the majority of the labs using the OpenShift CLI, but you can also accomplish them using the Azure Red Hat OpenShift web console.

## Prerequisites

## Login to the Azure Portal

1. In the LabVM, click on the Azure portal shortcut of the Microsoft Edge browser, which is created on the desktop.

   ![](../media/vm30.png)

1. On the **Sign in to Microsoft Azure** tab, you will see the login screen. Enter the following email or username, and click on **Next**.

   - **Email/Username:** <inject key="AzureAdUserEmail"></inject>
 
     ![Enter Your Username](../media/login1.png)

1. Now enter the following password and click on **Sign in**.
 
   - **Password:** <inject key="AzureAdUserPassword"></inject>
 
      ![Enter Your Password](../media/login2.png)

1. If you see the pop-up **Stay Signed in?**, select **No**.

1. If you see the pop-up **You have free Azure Advisor recommendations!**, close the window to continue the lab.

1. If a **Welcome to Microsoft Azure** popup window appears, select **Maybe Later** to skip the tour.

#### Azure Cloud Shell

1. In the Azure portal, open the **Azure Cloud Shell** by clicking on the icon in the top right of the Azure Portal.

    ![Image](../media/cloudshell.png)

1. When prompted to select either **Bash** or **PowerShell**, select **PowerShell**. 

    >**Did you know?**  If you mostly work with Linux systems, Bash (CLI) feels more familiar. If you mostly work with Windows systems, Azure PowerShell feels more familiar. 

1. On the **You have no storage mounted** screen select **Mount Storage Account**, select the storage account subscription from drop down and click on **Apply**

1. On mount storage account page, select **I want to create a storage account**. click on Next.

1. Provide the below details to create the storage account and click on **Create**.

    >**Note:** As you work with the Cloud Shell a storage account and file share is required. 

    | Settings | Values |
    |  -- | -- |
    | Resource Group | **openshift** |
    | Storage account (Create new) | **str<inject key="DeploymentID" enableCopy="false" />** |
    | Region | **<inject key="Region" enableCopy="false" />** |
    | File share (Create new) | **none** |

1. Wait for the deployment to succeed and you should now have access to the Azure Cloud Shell

#### OpenShift CLI (oc)

You'll need to download the **latest OpenShift CLI (oc)** client tools for OpenShift 4. You can follow the steps below on the Azure Cloud Shell.

> **Note** You'll need to change the link below to the latest link you get from the page.
> ![GitHub release links](../media/github-oc-release.png)

Please run following commands on Azure Cloud Shell to download and setup the OpenShift client.

```sh
cd ~
curl https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz > openshift-client-linux.tar.gz

mkdir openshiftdir

tar -zxvf openshift-client-linux.tar.gz -C openshift

echo 'export PATH=$PATH:~/openshift' >> ~/.bashrc && source ~/.bashrc

```

The OpenShift CLI (oc) is now installed.

In case you want to work from your own operating system, here are the links to the different versions of CLI:

- https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-windows.zip
- https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
- https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-mac.tar.gz

#### GitHub Account
You'll need a personal GitHub account. You can sign up for free [here](https://github.com/join).

## 1.2 Basic concepts

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
