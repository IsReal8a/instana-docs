---
title: IBM Kubecost integration
parent: Integrations
nav_order: 3
---

# IBM Instana and IBM Kubecost integration
{: .no_toc }

Technical guide on how to integrate IBM Instana with IBM Kubecost, this approach is using the Instana agent running in one RedHat OpenShift cluster and Kubecost free tier.
With some slight changes, it should work for other implementations but for RedHat OpenShift it was a bit harder than expected.
{: .fs-6 .fw-300 }

Official documentation

[From IBM Instana](https://www.ibm.com/docs/en/instana-observability/1.0.293?topic=apis-integrating-kubecost-public-preview){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[From IBM Kubecost](https://www.ibm.com/docs/en/kubecost/self-hosted/2.x?topic=installation){: .btn .fs-5 .mb-4 .mb-md-0 }

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## IBM KubeCost configuration
### Install/Upgrade KubeCost

You need to install IBM Kubecost on the same platform where you have the Instana agent running, this guide is going to help you integrate an Instana agent running in RedHat OpenShift.

If you subscribe to the IBM Kubecost Free tier then you will get this page:

[Install IBM KubeCost](https://www.kubecost.com/install#show-instructions){: .btn }

However, that may not be what you're looking for, because, you may have another provider like OpenShift:

[Provider Installations](https://www.ibm.com/docs/en/kubecost/self-hosted/2.x?topic=installation-provider-installations){: .btn }

I'm going to guide you how to install IBM Kubecost using the Standard deployment:

Read the below first to get familiar with the documentation:

[IBM Kubecost Standard deployment guide](https://www.ibm.com/docs/en/kubecost/self-hosted/2.x?topic=installations-install-kubecost-red-hat-openshift#standard-deployment-guide){: .btn }

In a host with connectivity to your OpenShift cluster (assuming you're logged-in), you need to run the following commands:

```shell
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm repo update
```

Install IBM Kubecost on RedHat Openshift

{: .warning }
> It's imperative to install it with the same `clusterName` defined in the Instana agent configuration, for example, I defined the `clusterName` in Instana as `CP4BA_IO`, the same `clusterName` needs to be used in Kubecost.

```shell
helm upgrade --install kubecost kubecost/cost-analyzer -n kubecost --create-namespace \
-f https://raw.githubusercontent.com/kubecost/cost-analyzer-helm-chart/v2.5/cost-analyzer/values-openshift.yaml \
--set kubecostProductConfigs.clusterName=CP4BA_IO \
--set prometheus.server.global.external_labels.cluster_id=CP4BA_IO
```

### How to access the IBM Kubecost UI

Do a port forwarding as shown below and open http://localhost:9090 in your browser.

```shell
kubectl port-forward --namespace kubecost deployment/kubecost-cost-analyzer 9090
```

![Kubecost UI](image.png)

### Delete IBM Kubecost

Did you mess up somehow or don't need Kubecost anymore?

Delete IBM Kubecost!

```shell
helm uninstall kubecost -n kubecost
```

## IBM Instana configuration

Now that you have IBM Kubecost up and running you need to configure the Instana agent to communicate with Kubecost, you can go and use the official documentation but that doesn't work for RedHat OpenShift:

[Instana and Kubecost](https://www.ibm.com/docs/en/instana-observability/current?topic=apis-integrating-kubecost-public-preview){: .btn }

### Look for the IBM Kubecost "URL"

Well, IBM Kubecost doesn't expose that to the world it seems but it's being shared by the `kubecost-cost-analyzer` deployment, lets use Instana to get that.

Go to the left hand side menu and click in "Platforms"-> "Kubernetes" and search for your cluster, once there, go to the `kubecost` namespace then `kubecost-cost-analyzer` deployment, something like this:
![Kubecost cost analyzer deployment](image-1.png)

There, in that example, you can see that the service is located in `172.30.192.31` on port `9090`, that's the one we need to continue.

### Build your IBM Instana agent CR YAML and apply it

You need to push the new configuration to the Instana agent (or install it for first time if you haven't), remember the `clusterName` needs to be the same in both places, plus, you need to use the same name in the `clusters` value inside the Kubecost plugin, you can use the following Instana agent configuration as an example and modify it to match your values, pay attention to the `url` and the `port` in the configuration as this is the IP address and port I got from the previous section, if you don't do all this, you won't see a thing in the Instana UI:

```shell
apiVersion: instana.io/v1
kind: InstanaAgent
metadata:
  name: instana-agent
  namespace: instana-agent
spec:
  zone:
    name: DarkZone # (optional) name of the zone of the host
  cluster:
      name: CP4BA_IO
  agent:
    key: AGENT_KEY
    downloadKey: DOWNLOAD_KEY
    endpointHost: ingress-orange-saas.instana.io
    endpointPort: "443"
    env: {}
    configuration_yaml: |
      com.instana.plugin.kubecost:
        remote:
          - url: '172.30.192.31:9090'
            poll_rate: 1800 # seconds
            clusters: #List of one or more k8s cluster names.
            - 'CP4BA_IO'
```

Save the file as `instana-agent-cr.yaml` and apply it:

```shell
kubectl apply -f instana-agent-cr.yaml
```

If everything went well, on the Instana UI, go to the left hand side menu, click "Platforms"-> "Kubernetes", search for your Cluster and at the end you're going to see the tab "Cost" click on it and you should have something similar as this:

![Instana Kubecost info](image-2.png)

If you do, that's it!

## Exploring uncharted territory

So far, with this installation, you get Prometheus and Grafana with KubeCost, but you may not need to access them... what if we use the Prometheus endpoints to send the data to Instana and then use the Instana UI for our purposes...
That's another chapter.