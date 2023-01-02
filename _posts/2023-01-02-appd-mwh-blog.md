---
title: Monitoring Kubernetes workloads with AppDynamics and modern CD strategies 
---
 
Monitoring Kubernetes workloads with AppDynamics and modern CD strategies 
-------------------------------------------------------------------------

Many organizations move their applications onto cloud native infrastructure like Kubernetes or its’ “superstructures” like OpenShift or Rancher, both on premises and in the public clouds.  

AppDynamics as a leading APM (Application Performance Monitoring) tool always offered support for workloads in those environments and aided monitoring needs from multiple angles. It offers platform monitoring, individual pod monitoring and deep application performance monitoring via specialized agents. While platform monitoring and pod monitoring is handled with one-time setup of a Cluster Agent, for APM, agents need to be embedded into application pods and there are multiple ways to do it – but there are a few specifically desired properties of the process in practice: 

- The process of embedding an agent should be easy from the developer standpoint, ideally transparent 

- The process should work well in context of CI/CD pipelines used 

- The method should be robust and work with all Kubernetes resource kinds like Deployments, StatefulSets, DaemonSets, Jobs, CronJobs, etc. 

- The method should not require elevated privileges for the process of embedding 

AppDynamics Cluster Agent provides such process called auto-instrumentation feature for Java, .NET, and Node.js applications out of the box, and while it works well, there are certain limitations, especially when using increasingly popular GitOps continuous deployment tools like Argo CD. Those tools compare application specification in the git with the running application and when there are differences, git version wins and is automatically adjusted on the Kubernetes cluster.  

Because AppDynamics Cluster Agent embeds the agents via dynamic application definition changes, tools like Argo CD detect that and revert the changes to the state defined in the git. Cluster Agent sees the application ready for instrumentation, does the injection again – and you get the idea by now – this process can go on forever. Also, Cluster Agent does not support some workload types like CronJobs.  

![](<../images/appd-cluster-agent-auto-instr.drawio.svg>)

Img – AppDynamics Cluster Agent auto-instrumentation process direct application deployment

![](<../images/appd-cluster-agent-auto-instr-argo.drawio.svg>)

Img – AppDynamics Cluster Agent auto-instrumentation process with Argo CD

Having multiple customers using AppDynamics and facing those or similar issues, I have been looking for a different way of the entire process. After some experimenting, I decided to approach the problem using a standard Kubernetes extension feature – mutating webhook. Mutating webhooks can be registered to certain resource events in the cluster, among many other events, to pod instantiation event. Then they can implement functionality, which modifies the resource -  pod in our case - before it is created. This avoids the conflicts described above and works well with any kind of resource (except direct Pod deployment via Argo CD, for example, which is rare these days). As a result, I have developed a fully functional solution, published as an open-source project with Cisco Systems permission, and it is currently running in production at scale at multiple customers.  

![](<../images/appd-mwh-auto-instr-argo.drawio.svg>)

Img – Mutating Webhook-based auto-instrumentation process 

If interested, feel free to use yourselves and here is how.  

How To
------

Clone the repo: 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
git clone https://github.com/cisco-open/appdynamics-k8s-webhook-instrumentor.git 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now we need to specify instrumentation rules. Since the instrumentation webhook is deployed using Helm, this is done by creating a file with values for Helm chart. 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
cd appdynamics-k8s-webhook-instrumentor/buildEnv/helm
cp values-sample.yaml my-values.yaml
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In this example, we're going to use AppDynamics native agents and native telemetry feed only, though we support AppDynamics agents hybrid mode with OpenTelemetry traces feed and also native OpenTelemetry agents - this I can cover in subsequent blogs. 

Open `my-values.yaml` in your favourite text editor.

1. Find section appdController - fill in appropritate values
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
appdController:
  host: <instance>.saas.appdynamics.com
  port: "443"
  isSecure: true
  accountName: <instance>
  accessKey: "xxxxxxxxxxx"
  # following is optional - use only if using proxy to access SaaS controller
  # useProxy: true
  # proxyHost: proxy.corm.com
  # proxyPort: "8080"
  # following is optional - use only if using hybrid agent mode
  # otelEndpoint: https://<region>-sls-agent-api.saas.appdynamics.com 
  # otelHeaderKey: "xxx" 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

2. Delete `openTelemetryCollectors:` section if not using OpenTelemetry and hybrid agent mode
   
3. Specify instrumentation template for your languages in `instrumentationTemplates` section
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
instrumentationTemplates:
  - name: Java_Default
    injectionRules:
      # technology = java | dotnetcore | nodejs 
      technology: java
      image: appdynamics/java-agent:latest
      javaEnvVar: JAVA_TOOL_OPTIONS
      applicationNameSource: label
      applicationNameLabel: appdApp
      tierNameSource: auto
      # optional otelNativeAgent: true -> assume native otel image use, no AppD native feed
      # openTelemetryCollectors: deploymentHybridagentDefault
  - name: Dotnet_Default
    injectionRules:
      technology: dotnetcore
      image: appdynamics/dotnet-core-agent:22.5.1
      applicationNameSource: label
      applicationNameLabel: appdApp
      tierNameSource: auto
  - name: Nodejs_Default
    injectionRules:
      technology: nodejs
      image: appdynamics/nodejs-agent:22.5.0-16-stretch-slim
      applicationNameSource: label
      applicationNameLabel: appdApp
      tierNameSource: auto
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

AppDynamics agent needs a few parameters - at least application name and tier name. `applicationNameSource` attribute tells where to get the application name from. Here we say we take it from Pod label abd the label is `appdApp` as specified by `applicationNameLabel` attribute. `tierNameSource` attribute set to `auto` means that tier name will be derived from resources owning the Pod, which ends up being Deployment or StatefulSet name, for example. 

Pls. note we always specify `technology` attribute which simply tells what framework the application uses and which agent type therefore to use. Agent is then delivered by specified `image`.

4. Specify selection rules for auto-instrumentaion

