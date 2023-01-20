---
title: AppDynamics, OpenTelemetry and Auto-instrumentation 
---

OpenTelemetry is considered a lingua franca of the future for telemetry collection. There are excellent reads on what OpenTelemetry is, so I am not going to do it again, if interested anyway, I suggest reading following: 

https://opentelemetry.io/docs/concepts/what-is-opentelemetry/ 

https://opentelemetry.io/docs/collector/ 

https://www.appdynamics.com/blog/product/what-is-opentelemetry/ 

 

AppDynamics understands the importance and promises of OpenTelemetry very well, after all, the all new AppDynamics Cloud solution is based on OpenTelemetry signal feeds. Since early 2022, traditional AppDynamics cSaaS (the traditional platform) supports OpenTelemetry, too, though for the time being for traces only.  

 

Being an enthusiastic fan of OpenTelemetry for about 2 years now, I started to play with AppDynamics cSaaS and different setups of applications instrumented with OpenTelemetry compliant agents since the initial support. It does not sound complicated at the first look, but there are quite a few options and lots of combinations to try: 

- Agent – AppDynamics Hybrid agents can send native data feed to AppDynamics controller, but also OpenTelemetry feed to OpenTelemetry collector and from there, to any tracing backend application. There are also native OpenTelemetry agents and other 3rd party agents sending tracing data. They are not equal in their capabilities in terms of instrumented frameworks and libraries and in richness of collected data. The first question therefore is – which agent to use? 

- OpenTelemetry Collector – a component not found in traditional architectures of APM (Application Performance Monitoring) tools, where agents typically connect to backends directly. In OpenTelemetry case, it is needed, and the question is, where it should be located – on application server itself dedicated for a single agent? On separate machine serving multiple agents? On container platforms, should it be a sidecar container in a pod? Should it run on the cluster or elsewhere? How about scalability and reliability? 

- The telemetry backend – in my case, of course, AppDynamics cSaaS, but why not try it with Jaeger or Zipkin? And since the collector can split the feed to multiple platforms, why not use that? 

![](<../images/mwh-otel-1/agt-coll-bck-arch.drawio.svg>)

Img 1. -  Architectural options with agents, collectors, and different protocols

 
The best learning experience for me is always hands-on. Therefore, I collected some sample applications in Java, .NET / C#, and Javascript / Node.js and started to use different agents, different collector configurations, different backends, and quite quickly, it got completely out of hand – so many combinations, startup scripts, etc. 

 
Surely, there must be a better way, I thought. Finally, I decided to use Kubernetes to run sample applications as microservices and to expand capabilities of my auto-instrumentation tool for AppDynamics agents described in my [previous blog](</_posts/2023-01-02-appd-mwh-blog.md>) by an ability to also inject another agents, configure them, and run OpenTelemetry collectors in different setups and configurations. Then, each lab scenario is simply described by single `values.yaml` file for Helm deployment, while the application workloads stay the same. Much easier. 

## Interested in trying yourselves? 

We are going to start with a simple scenario today – one Java microservice, OpenTelemetry collector, and connect it to AppDynamics cSaaS controller. The following diagram shows what we are going to try to achieve. 

![](<../images/mwh-otel-1/agt-appd-hybrid.drawio.svg>) 

Img 2. - AppDynamics Hybrid Agent Architecture (Java)

The configuration enabling OpenTelemetry usage is highlighted in green and the diagram shows only the OpenTelemetry related data path, not the native AppDynamics telemetry feed. That goes, as always, from the (Java) agent to AppDynamics controller.

First, follow the download / clone instructions from my previous blog. Once done, download `values-sample-otel.yaml` file from [this gist](<https://gist.github.com/chrlic/967fb9308bd778e570e91a11f7f467f4#file-values-sample-otel-yaml>) and fill in AppDynamics controller connection parameters, including the otelEndpoint and otelHeaderKey values.  

 

The `otelEndpoint` value is specific per location of your AppDynamics cSaaS controller and can be found here: https://docs.appdynamics.com/appd/23.x/23.1/en/appdynamics-essentials/getting-started/saas-domains-and-ip-ranges 

 

The `otelHeaderKey` value is specific to your instance and is located on the bottom of the OTel tab in the controller GUI. 

Look for <…> placeholders to find what needs to be configured. 

Now it is time to look at OpenTelemetry collector definition under the `openTelemetryCollectors` section. Each key here represents an OpenTelemetry collector later used in instrumentation rules. In the sample file, you can see three sample definitions with different architectures: 

- collector running as a separate Deployment. It will be deployed by the helm chart. 

- collector running as a sidecar container in individual Pods, which is done alongside with the agent injection automatically 

- Collector running as an unmanaged entity – on the Kubernetes or elsewhere. We just assume it exists. 

 
The following diagram shows OpenTelemetry collector as a standalone deployment/pod, potentially shared by multiple applications in multiple namespaces. It shows in green what is done by the auto-instrumentation process automatically. 

 
![](<../images/mwh-otel-1/k8s-agent-arch-depl.drawio.svg>)
Img 3. - Using Standalone OpenTelemetry Collector 

 
Sometimes, it might be beneficial to have an OpenTelemetry collector effectively part of the application, for example, when specific configuration, like batching, sampling, service naming, or other pre-processing of telemetry information is required. The following diagram shows this scenario, where collector runs as a sidecar container in the same pod as an application itself. Again, configuration made by auto-instrumentation is shown in green. 

 
![](<../images/mwh-otel-1/k8s-agent-arch-side.drawio.svg>)
Img 4. - Using OpenTelemetry Collector as a Sidecar 

 

The specific architecture used is defined by collector `mode` in the Help values file – deployment, sidecar, or external. 

 

As you can see, the definition contains templated configuration of OpenTelemetry collector, where appropriate values are going to be filled in by the Helm chart for our convenience.  

 

When using AppDynamics hybrid agent, all we must do now, is to include `openTelemetryCollector: sidecar-hybrid-agent-default` in the `instrumentationTemplates` and refer to the template in the `injectionRules`, so it looks something like this: 

 

``` 
instrumentationTemplates:
  - name: Java_Appd_Otel
    injectionRules:
      # technology = java | dotnetcore | nodejs
      # provider = appd | otel - appd is default if missing
      technology: java/appd
      image: appdynamics/java-agent:latest
      javaEnvVar: JAVA_TOOL_OPTIONS
      applicationNameSource: label
      applicationNameLabel: appdApp
      tierNameSource: auto
      openTelemetryCollector: sidecar-hybrid-agent-default

instrumentationRules:
  - name: java-otel-test
    matchRules:
      namespaceRegex: .*
      labels:
      - otel: appd
      - language: java
      annotations:
      - annot1: .*
      podNameRegex: .*
    injectionRules:
      template: Java_Appd_Otel
 
``` 

 

Now, start the mutating webhook instrumentation via Helm – details in the [last blog](</_posts/2023-01-02-appd-mwh-blog.md>), step #5 

` kubectl create namespace mwh` # if you do not have the namespace yet 

`helm install --namespace=mwh mwh . --values= values-sample-otel.yaml`  

Check that the webhook is running: 

`kubectl –n mwh get pods` 

 

Now it is time to deploy some workload – let us use the same application as last time, but slighly different [deployment manifest](<https://gist.github.com/chrlic/967fb9308bd778e570e91a11f7f467f4#file-d-downstream-yaml>)
 

Once the workload starts, you can test it:

`curl http://localhost:8383/downstream/hello`

However, since we are now using AppDynamics Java agent in hybrid mode, there are two more things to verify. 

 

First, let us have a look if the OpenTelemetry collector started as a sidecar collector: 

`kubectl get pod <pod-name> -o jsonpath='{range .items[*]}{"\n"}{.metadata.name}{"\t"}{.metadata.namespace}{"\t"}{range .spec.containers[*]}{.name}{"=>"}{.image}{","}{end}{end}'|sort|column` 

where <pod-name> is the name of the application pod (something like `downstream-6f8fb4d457-m2gzk`)


Second, verify that the OpenTelemetry collector started successfully by looking at the log. Since the collector log also contains log exporter with debug enabled, you will see the traces processed by it. Do not use this in production, but here we are in a lab. 

`kubectl logs –f <pod-name> -c otel-coll-sidecar` 

Second, once you send some test calls via curl or browser, AppDynamics will create two applications now – one as the native AppDynamics application and a second one with _otel suffix, which is focused more on data ingested via OpenTelemetry. Please take into consideration that for the first time, it takes a bit more time than usual before the _otel application shows up. You should see something like this:

 
![](<../images/mwh-otel-1/appd-otel-apps.png>)
Img 5. - AppDynamics Applications View 

 

Did you make it here? Congratulations!


Feel free to experiment, like changing `sidecar-hybrid-agent-default` to ` deployment-hybrid-agent-default`, and restart the instrumentation: 

`helm uninstall --namespace=mwh mwh` 

`helm install --namespace=mwh mwh . --values= values-otel1.yaml` 

 
Now, restart the application. What differences do you see in topology on the Kubernetes? Any difference on AppDynamics? 


Next time, we will have a look into more complex scenarios.  

 

 

  