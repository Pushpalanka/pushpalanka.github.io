---
title: "SPIFFE in a Nutshell"
date: 2020-04-28T00:00:00Z
draft: false
author: "Pushpalanka"
tags: ["spiffe", "spire", "security", "identity", "cloud-native"]
categories: ["Technical"]
description: "A high-level architectural flow and conceptual breakdown of the Secure Production Identity Framework For Everyone (SPIFFE) and SPIRE."
cover:
  image: "/images/spiffe/spiffe-flow.webp"
  alt: "SPIFFE Architecture Flow Illustration"
canonicalURL: https://pushpalanka.com/posts/spiffe-in-a-nutshell
---

I have been studying SPIFFE (Secure Production Identity Framework For Everyone) [1] for some time and here I am drafting the flow as I have understood now, for the benefit of anyone else trying to understand the flow.

### Core Concepts

*   **Identity Registry** — The SPIRE server(A SPIFFE implementation) has its own identity registry which keeps two coarse-grained attributes that decide how the SPIFFE IDs will be issued to a workload. A separate registration API is provided to manage these entries in the identity registry.
*   **Node Selector** — This defines a machine (physical or virtual) where a workload can be running on. The exact type of selector to be used is decided based on the infrastructure provider (AWS, GCP, bare metal) that the workload is running on. *E.g., AWS EC2 Instance ID, or the serial number of a physical machine.* Node attestors act based on the infrastructure provider to honor their selectors.
*   **Workload Selector** — This defines how to identify a process as representing a workload, after the node is identified. This can be described in terms of attributes of the process itself (e.g., Linux UID) or in terms of indirect attributes such as a Kubernetes namespace. The node agent is responsible for verifying that a particular process on a machine qualifies for its workload selector. Workload attestors act based on the process attributes to honor the process selectors.
*   **SPIRE Node Agent** — A process that sits on the node, verifies the provenance of workloads running on the node, and provides those workloads with certificates via the Workload API, based on the selectors.

---
{{< figure src="/images/spiffe/spiffe-flow.webp" caption="SPIFFE/SPIRE Architecture Communication Flow" width="800" align="center" >}}

### Architecture & Communication Flow

1. The **Registration API** is called by either an administrator or a third-party application to populate the identity registry with the required SPIFFE IDs and relevant selectors.
2. The **Node Agent** gets authenticated with the SPIRE server using a pre-established cryptographic key pair or based on the infrastructure provider. For example, in the case of AWS EC2, the node agent will submit the node’s Instance Identification Document (IID) issued by AWS.
3. The **Node Attestor** in the SPIRE server validates the provided identification document based on the mechanism used. If the AWS IID is used, the relevant attestor will validate it against AWS settings. Upon successful validation, the SPIRE server sends back a set of SPIFFE IDs that can be issued to the node along with their process selector policies.
4. When a **workload** starts to run on the node, it first makes a call to the node agent asking, *"Who am I?"*.
5. Based on the process selectors the node agent received in the previous step, and using the workload attestors, the **agent decides on the SPIFFE ID** to be given to the workload. It generates a key pair based on that and sends a Certificate Signing Request (CSR) to the SPIRE server.
6. The **SPIRE server responds** to the node agent with the signed SVID for the workload along with the trust bundles, indicating which other workloads can be trusted by this workload.
7. Upon receiving the response from the SPIRE server, the **node agent hands over** the received SVID, trust bundles, and the generated private key to the workload. This private key never leaves the node to which its workload belongs.

Cheers!

---

### References

*   [1] — [SPIFFE Official Site](https://spiffe.io)
*   [2] — [SPIRE Concepts Document](https://docs.google.com/document/d/1RZnBfj8I5xs8Yi_BPEKBRp0K3UnIJYTDg_31rfTt4j8/edit#)
*   [3] — [Authorization for Multi-Cloud System](https://pushpalankajaya.blogspot.com/2018/12/authorization-for-multi-cloud-system.html)

*Originally published at pushpalankajaya.blogspot.com*