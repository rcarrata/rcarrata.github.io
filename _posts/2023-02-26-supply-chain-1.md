---
layout: post
title: Securing the integrity of Software Supply Chains
date: 2023-02-26
type: post
published: true
status: publish
categories:
- Kubernetes
- Security
tags: []
author: rcarrata
comments: true
---

## Overview

How can we secure the integrity of our Software Supply Chains and have confidence that our software has not been tampered with and can be traced to its source? Which are the main parts of the software supply chain security?

Before explaining how to secure the Software Supply Chain, we need first to understand what it is and why it matters to our environments and application deployments.

A software supply chain is the collection of entities, processes, policies, services and infrastructure that have some involvement over any stage of the delivery of software to a customer. A service or a CI/CD pipeline is a subset of a supply chain.

So, in a nutshell, a software supply chain is the series of steps performed when writing, testing, packaging and distributing application software to end consumers. It is the process of getting a product to a customer.

NOTE: this blog post was original posted in [Opensourcerers](https://www.opensourcerers.org/2022/11/21/securing-the-integrity-of-software-supply-chains/).

## The Rise of Supply Chain Attacks

The [Sunburst supply chain](https://www.opensourcerers.org/2022/11/21/securing-the-integrity-of-software-supply-chains/) compromise was a hostile intrusion of Fortune-500 and US and UK Government networks through malware hidden in a signed legitimately, but compromised server monitoring agent.

The Cozy Bear hacking group managed to hack the update server of Solarwinds, a widely used and trusted network monitoring software, compromising over 18000 organizations, including billion dollar companies, simultaneously. It’s speculated that the recovery may cost upward of $100B. 

SUNBURST attacks infected SolarWinds CI/CD pipelines and modified and altered source code before it was built, hiding the evidence of tampering and ensuring that the binary was signed by the CI/CD pipelines so consumers would trust this binary without hesitation. It has been described as the largest and most sophisticated supply chain attack the world has ever seen.

The SUNBURST supply attacks in 2020 directed the attention on the importance of how software is built, and what vulnerabilities are potentially left open by that process. 

But this was not the only Supply Chain Attack in history, examples such as Kaseya Ransomware Attack, [CodeCov](https://blog.gitguardian.com/codecov-supply-chain-breach/) or [Log4j Vulnerability](https://www.cisa.gov/uscert/apache-log4j-vulnerability-guidance) among others, are also excellent examples of supply chain attacks affecting thousands of systems and companies, causing inimaginable losses of data and money.

## OpenSSF and CNCF Tag-Security

With the rise of these and other supply chain attacks, there are several organizations and groups that help and work actively to improve the security in the supply chain landscape.

Founded in 2020, the [Open Source Security Foundation](https://openssf.org/) (OpenSSF) has begun to devise improved defenses against software supply chain attacks.

The [Sigstore project](https://www.sigstore.dev/) is one of these improved defenses, providing a method for guaranteeing the end-to-end integrity of software artifacts.

Furthermore, the [CNCF Security Technical Advisory Group](https://github.com/cncf/tag-security) (STAG or Tag-Security) facilitates collaboration to discover and produce resources that enable secure access, policy control, and safety for operators, administrators, developers, and end-users across the cloud native ecosystem.

## Types of Supply Chain Attack

Not all attacks towards the Supply Chain are the same, and we can categorize these in several types to tackle and secure each of these types of attacks.

The CNCF Security Technical Advisory Group (STAG) published a list of [Types of Supply Chain Compromises Definitions](https://github.com/cncf/tag-security/blob/main/supply-chain-security/compromises/compromise-definitions.md), providing a consistent set of definitions of each compromise type.

Also the STAG published and maintains a [Catalog of Supply Chain Compromises](https://github.com/cncf/tag-security/tree/main/supply-chain-security/compromises), capturing many examples of different types of attacks, helping to understand the patterns and developing in that way best practices and tools to defend against.

Finally the [National Cyber Security Centre from the UK Government](https://www.ncsc.gov.uk/collection/supply-chain-security/supply-chain-attack-examples) provides guidance to identify the types of supply chain attacks examples.

## Securing cloud native systems from Supply Chain Attacks

At a high level, supply chain attacks are relatively simple: impact trusted parts of your system that we usually do not observe with caution, like the CI/CD patterns of third party suppliers.

The following diagram depicts how the software supply chain closely mirrors a traditional manufacturing supply chain, with some notable differences such as intangibility, iterative usage and processes that continually change: 

[![](/images/supply_chain1.png "supply_chain1.png")]({{site.url}}/images/supply_chain1.png)

But securing the supply chain problem is complex and extensive, making it difficult to protect an organization’s supply chain. Any stage in the supply chain that is not under your direct control is liable to be attacked, and a successful supply chain attack is often difficult to detect, because the consumer usually trusts every producer in their supply chain.

We need to understand that the software factory generally creates multiple pipelines configured to build a software binary from artifacts to be deployed to several systems in multiple environments. 

These pipelines are composed from individual steps or build stages that are chained using tools like Jenkins or Tekton (among others) with the purpose of:

* Retrieve the source code
* Retrieve the third parties dependencies
* Build the artefact
* Deploy the artefact

To eliminate the chance of human errors in these steps within the pipelines, there shouldn’t be any manual steps but rather immutable pipelines leading to automation and embracing the infra and security-as-code.

As explained in the [CNCF STAG Security Supply Chain Reference Architecture](https://github.com/cncf/tag-security/blob/main/supply-chain-security/secure-software-factory/secure-software-factory.md) the key principles for supply chain security are:

1. **Trust**: Every step in a supply chain should be “trustworthy” due to a combination of cryptographic attestation and verification.
2. **Automation**: Automation is critical to supply chain security and can significantly reduce the possibility of human error and configuration drift.
3. **Clarity**: The build environments used in a supply chain should be clearly defined, with limited scope.
4. **Mutual Authentication**: All entities operating in the supply chain environment must be required to mutually authenticate using hardened authentication mechanisms with regular key rotation.

## Supply Chain Levels for Software Artifacts (SLSA)

The Open Source Security Foundation (OpenSSF) SLSA Framework (aka Salsa) defines a graded approach to adopt the supply chain for the artefact builds.

It’s a security framework, a check-list of standards and controls to prevent tampering, improve integrity, and secure packages and infrastructure in our projects, businesses or enterprises. 

SLSA is organized into a [series of levels](https://slsa.dev/spec/v0.1/levels) that provide increasing integrity guarantees. This gives us the confidence that software hasn’t been tampered with and can be securely traced back to its source.

[![](/images/supply_chain2.png "supply_chain2.png")]({{site.url}}/images/supply_chain2.png)

## Summary

In this blog post, we reviewed the importance of securing our Software Supply Chains, describing the most relevant attacks that happened in recent history and how to identify the types of supply chain attacks available.

Furthermore, we analyzed how we can secure our Software Supply Chains, and described the main parts of the Supply Chain security, identifying which parts we need to focus and secure.

Finally we introduced several groups that are actively helping in developing tools and frameworks to improve the defenses against software supply chain attacks such as OpenSSF or CNCF STAG, among others. Also we reviewed the SLSA framework, providing a series of levels to increase the confidence that our software has not been tampered and can be traced to its source.

In the next blog posts, we will analyze in depth the tools that can be used to help in one of each parts of the software supply chain security.

Happy securying!

Peace and Cosign!