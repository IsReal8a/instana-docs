---
title: Instana agent on real airgap
parent: Agent
nav_order: 3
---

# Instana agent on a Redhat Openshift cluster real airgap
{: .no_toc }

A couple scripts that can help you install the Instana agent on an OpenShift cluster isolated from the World.

{: .fs-6 .fw-300 }

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## What is the need?

{: .important }
> Some customers, mainly on the Banking sector, have a more restricted IT regulations, rules and processes, that includes having Redhat Openshift clusters not connected to anything that faces the Internet.

Therefore, I needed a way to copy all necessary files and configuraton into an "Instana agent offline package" that can be transported on a physical medium and then copy its contents inside the "real airgap" environment, then install the Instana agent.

Created a couple of scripts that can help on this.

[Take me to the scripts!](https://github.com/IsReal8a/instana-examples/tree/main/agent_real_airgap){: .btn .btn-blue}
