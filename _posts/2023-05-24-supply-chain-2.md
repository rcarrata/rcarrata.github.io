---
layout: single
title: Building Trust in the Software Supply Chain
date: 2023-05-24
type: post
published: true
status: publish
categories:
- security
- Kubernetes
- DevSecOps
tags: []
author: rcarrata
comments: true
---

What steps can we take to establish trust in our Software Supply Chain and ensure that our software can be traced back to its origin without introducing malicious code or dependencies? Moreover, how can we integrate Open Source tools to enhance the security of our Software Supply Chain’s lifecycle?

As we explained in our [first blog post](https://rcarrata.com/kubernetes/security/supply-chain-1/), software supply chain is the series of steps performed when writing, testing, packaging, and distributing application software to end consumers.

Establishing trust in the software supply chain has become essential to ensuring software components’ security and reliability. With the majority of open-source software and the growing demand for supply chain management, it’s crucial to have robust processes to prevent malicious code or dependencies from being introduced; integrating open-source tools can enhance the software supply chain’s lifecycle security. 

In this blog post that my friend and colleague [Rodrigo Alvares](https://www.linkedin.com/in/ralvares/) prepared, we will discuss the actions you can take to establish trust in the software supply chain and the critical role that open-source tools can play in verifying the reliability of software components.

NOTE: this blog post was original posted in [Opensourcerers](https://www.opensourcerers.org/2023/04/24/building-trust-in-the-software-supply-chain/) the 24th of April of 2023.

## 1. Distributed components of the Secure Software Factory

The software supply chain is a crucial process that involves multiple steps, including writing, testing, packaging, and delivering application software to end-users. 
With the growing occurrence of software supply chain exploits and attacks, the[ Cloud Native Computing Foundation (CNCF) Technical Advisory Group for Security](https://github.com/cncf/tag-security) has taken proactive steps by publishing a comprehensive whitepaper titled “Software Supply Chain Best Practices” adopting the Software Factory Model for designing a secure software supply chain: [The Secure Software Factory](https://github.com/cncf/tag-security/blob/main/supply-chain-security/secure-software-factory/Secure_Software_Factory_Whitepaper.pdf).

[![](/images/supply_chain4.png "supply_chain")]({{site.url}}/images/supply_chain4.png)

The Secure Software Factory relies on source code, which includes the human-readable representation of applications being developed, as well as any dependencies that are either built from source or interpreted instead of compiled. The source code for both the build pipelines (Pipeline-as-Code) and the infrastructure (Infrastructure-as-Code) are included. 

During the pipeline’s execution, various metadata documents are created, including test reports, vulnerability reports, software attestations and Software Bills of Material (SBOMs). These documents capture the state of the build that generated them. 

For instance, a vulnerability report includes CVEs that were known during the build, but its accuracy may decline as new vulnerabilities are identified and disclosed. Similarly, an SBOM represents the contents of a specific build and remains relevant for that build. However, if future builds have slightly different dependencies or version constraints, a new or updated SBOM must be generated.

A software attestation refers to a verified metadata statement regarding a software artifact or a set of artifacts. Its main purpose is to provide input to automated policy engines, like Binary Authorization and [in-toto](https://in-toto.io/).

The SSF assumes that source code is managed using version control systems such as Git, with an established review and testing process in place that is suitable for the repository’s needs and use cases.

As the primary input for the SSF, it is up to the users and operators to determine which programming languages to support, where to host the source code, and which testing and scanning tools to integrate. 

In a nutshell, we need to be able to check the origin of all of these artifacts, like source code and dependencies (among others like attestations, signatures, etc.) and to trace all the packages and artifacts of the components of our Software Supply Chain.

## 2. Secure Software Factory Artifacts

The output that the Secure Software Factory produces is known as a software artifact, which serves as the primary deliverable. This artifact can take various forms, such as binaries, software packages, container images, signatures, or attestations, and it is designed to be utilized by downstream users.

To validate the artifact’s origin, it must be accompanied by the appropriate metadata, securely stored in an artifact repository, and distributed using secure and well-understood channels.

The specifics of the artifact’s characteristics and the execution of these requirements may differ based on variables like a programming language, package type, and target platform(s). Therefore, the Secure Software Factory does not address these implementation details.

Let’s see how we can use Open Source tooling to generate and manage all of these distributed components for our Secure Software Factory Supply Chain.

### 2.1 Building an example Application Artifact

Petclinic is a Spring Boot application built using Maven or Gradle, that we will use as an example in this blog post. We will download the source code and then build the Java application artifacts (JARs) that will be used for running this application:

```md
$ git clone https://github.com/spring-projects/spring-petclinic.git
$ cd spring-petclinic
$ ./mvnw package
$ java -jar target/*.jar
```

Some of the best practices for the Artifacts are the following:

* Artifacts must be available to downstream consumers and securely stored. 
* Signatures for artifacts should also be stored such that they can easily be found and verified. 
* These signatures can be stored alongside the artifact for convenient discoverability and distribution or in a separate location.

### 2.2 Dependency-Check (SCA)

Dependency-Check is a Software Composition Analysis (SCA) tool that attempts to detect publicly disclosed vulnerabilities contained within a project’s dependencies. It does this by determining if there is a Common Platform Enumeration (CPE) identifier for a given dependency. If found, it will generate a report linking to the associated CVE entries.

After installing locally, we can run a dependency check in our folder where we downloaded the source code and built all the artifacts, exploring the project dependencies and generating a report:

```md
$ dependency-check –enableExperimental –scan .
[INFO] Checking for updates
[INFO] NVD CVE requires several updates; this could take a couple of minutes.
[INFO] Download Started for NVD CVE – 2002
[INFO] Download Complete for NVD CVE – 2002  (1117 ms)
[INFO] Processing Started for NVD CVE – 2002
[INFO] Processing Complete for NVD CVE – 2002  (3210 ms)
…
[INFO] Analysis Started
[INFO] Finished Archive Analyzer (2 seconds)
[INFO] Finished File Name Analyzer (0 seconds)
[INFO] Finished Jar Analyzer (0 seconds)
[INFO] Finished Central Analyzer (32 seconds)
[INFO] Finished Python Distribution Analyzer (0 seconds)
[INFO] Finished Node.js Package Analyzer (0 seconds)
[INFO] Finished Dependency Merging Analyzer (0 seconds)
[INFO] Finished Version Filter Analyzer (0 seconds)
[INFO] Finished Hint Analyzer (0 seconds)
[INFO] Created CPE Index (0 seconds)
[INFO] Finished NPM CPE Analyzer (1 seconds)
[INFO] Created CPE Index (0 seconds)
[INFO] Finished CPE Analyzer (1 seconds)
[INFO] Finished False Positive Analyzer (0 seconds)
[INFO] Finished NVD CVE Analyzer (0 seconds)
[INFO] Finished RetireJS Analyzer (0 seconds)
[INFO] Finished Sonatype OSS Index Analyzer (1 seconds)
[INFO] Finished Vulnerability Suppression Analyzer (0 seconds)
[INFO] Finished Known Exploited Vulnerability Analyzer (0 seconds)
[INFO] Finished Dependency Bundling Analyzer (0 seconds)
[INFO] Finished Unused Suppression Rule Analyzer (0 seconds)
[INFO] Analysis Complete (39 seconds)
[INFO] Writing report to: /Users/rcarrata/Code/Security/spring-petclinic/./dependency-check-report.html
```

Dependency-check works by collecting information about the files it scans (using Analyzers). The information collected is called Evidence; there are three types of evidence collected: vendor, product, and version. 

For instance, the JarAnalyzer will collect information from the Manifest, pom.xml, and the package names within the JAR files scanned. It has heuristics to place the information from various sources into one or more buckets of evidence.

If we open the report generated from the dependency-check execution we can see a summary of the dependencies and more interesting information around CVEs, Evidences, etc:

[![](/images/supply_chain5.png "supply_chain")]({{site.url}}/images/supply_chain5.png)

### 2.3 Dependency-Check – File Type Analyzers

[OWASP dependency-check](https://jeremylong.github.io/DependencyCheck/analyzers/index.html) contains several file type analyzers that are used to extract identification information from the files analyzed.

Due to that, it is not only analyzing Java applications or Jar artifacts, it can analyze many more file types:

[![](/images/supply_chain6.png "supply_chain")]({{site.url}}/images/supply_chain6.png)

### 2.4 Building the Container Image

Now it’s time to build our Container image!

We will use the Dockerfile provided to build the image using Podman or Docker in order to be able to push it to a container registry.

We can then run the image in distributed systems such as Kubernetes or OpenShift (or in other systems):

```md
$ podman build -t sprint-petclinic:v1.0 . -f .devcontainer/Dockerfile
[+] Building 61.1s (8/8) FINISHED
 => [internal] load build definition from Dockerfile                                                                                  
=> transferring dockerfile: 627B                                                                                                  
 => [internal] load .dockerignore                                                                                                    
=> transferring context: 2B                                                                                                       
 => [internal] load metadata for mcr.microsoft.com/vscode/devcontainers/java:0-17-bullseye                                            
 => [1/4] FROM mcr.microsoft.com/vscode/devcontainers/java:0-17-bullseye@sha256:8e63b81b6dc5fa4dc9ff0bb3b707dc643e7c9cb63b70f2fe61b
=> resolve mcr.microsoft.com/vscode/devcontainers/java:0-17-bullseye@sha256:8e63b81b6dc5fa4dc9ff0bb3b707dc643e7c9cb63b70f2fe61b9
….
 => exporting to image                                                                                                                2.1s
 => exporting layers                                                                                                     
 =>writing image sha256:ca660a5bf3f86a03550121b368f312194b3a39ccc9572602c9284ff42e109ebe                                         
 =>naming to docker.io/library/sprint-petclinic:v1.0   
```

Now that we have the image generated, check the Container Image ID and the version:

```md
$ podman images | grep sprint
sprint-petclinic                                              v1.0      ca660a5bf3f8   5 minutes ago   1.86GB
```

### 2.5 Syft

[Syft](https://github.com/anchore/syft) is a CLI tool and Go library for generating a Software Bill of Materials (SBOM) from container images and filesystems.

Syft is compatible with various widely-used package formats in the most popular operating systems and programming languages. The list includes

* APK (Alpine), DEB (Debian), and RPM (Fedora) OS packages.
* Identification of Linux distributions across Alpine, CentOS, Debian, and RHEL favors.
* Go modules
* Java inJAR, EAR, and WAR variations
* NPM and Yarn packages
* Python Wheels and Eggs
* Ruby bundles

While not all programming languages are included, you can still take advantage of the OS-level scanning regardless of the technology stack used by your application.

To generate an SBOM for a container image:

```md
$ syft sprint-petclinic:v1.0
 ✔ Loaded image
 ✔ Parsed image
 ✔ Cataloged packages      [270 packages]

NAME                       VERSION                         TYPE
adduser                    3.118                           deb
apt                        2.2.4                           deb
apt-transport-https        2.2.4                           deb
apt-utils                  2.2.4                           deb
base-files                 11.1+deb11u6                    deb
base-passwd                3.5.51                          deb
bash                       5.1-2+deb11u1                   deb
binutils                   2.35.2-2                        deb
binutils-common            2.35.2-2                        deb
binutils-x86-64-linux-gnu  2.35.2-2                        deb
bsdextrautils              2.36.1-8+deb11u1                deb
…
```

The default output format is called table. It renders a columnar-based table of results in your terminal, creating a new row for each detected package:

```md
syft sprint-petclinic:v1.0 -o json > /tmp/sprint-petclinit-sbom-v1.0.json
 ✔ Loaded image
 ✔ Parsed image
 ✔ Cataloged packages      [270 packages]
```

With Syft, you can extract lists of packages from your container images, which provide you with an SBOM for your image. This generated data enhances your understanding of the length of your supply chain.

We can check the output of the SBOM json file generated by Syft:

```md
$ head /tmp/sprint-petclinit-sbom-v1.0.json
{
 “artifacts”: [
  {
   “id”: “3e9282034226b93f”,
   “name”: “adduser”,
   “version”: “3.118”,
   “type”: “deb”,
   “foundBy”: “dpkgdb-cataloger”,
   “locations”: [
    {
```

By incorporating Syft scans into your workflow, you will be kept updated on the packages you are utilizing. This will enable you to evaluate each package to determine its necessity. In case you come across numerous packages that are not essential for your workload, it is advisable to switch to a minimal base image and only add crucial software layers on top.

But wouldn’t it be amazing to have a user interface that could compile all dependencies, analyze SBOMs and other materials, and track and check for any CVEs that may affect our supply chain?

### 2.6 Dependency Track

[Dependency-Track](https://dependencytrack.org/) is an intelligent [Component Analysis](https://owasp.org/www-community/Component_Analysis) platform that allows organizations to identify and reduce risk in the software supply chain. Dependency-Track takes a unique and 

highly beneficial approach by leveraging the capabilities of [Software Bill of Materials](https://owasp.org/www-community/Component_Analysis#software-bill-of-materials-sbomsu) (SBOM). 

This approach provides capabilities that traditional Software Composition Analysis (SCA) solutions cannot achieve.

[![](/images/supply_chain7.png "supply_chain")]({{site.url}}/images/supply_chain7.png)

Dependency-Track monitors component usage across all versions of every application in its portfolio in order to proactively identify risk across an organization. The platform has an API-first design and is ideal for use in CI/CD environments.

After uploading the SBOM to the Dependency Track project (we created one called Secure Supply Chain Demo), we can see all the packages that are within the container image:

[![](/images/supply_chain7.png "supply_chain")]({{site.url}}/images/supply_chain7.png)

Alongside, we can see the Risk Score and the Vulnerabilities that affects each layer of our software, and therefore the risks that could be affecting our Supply Chain.

Furthermore, we will have a Project-Wide dashboard with the Overview of our demo Artifact and software:

[![](/images/supply_chain8.png "supply_chain")]({{site.url}}/images/supply_chain8.png)

## 3. Next Steps: Automate and Signing Images and Metadata Artifacts
Now that we have discussed the Open Source tools that we can use in our Secure Supply Chain, we need to move to the next step of the key principles of the Supply Chain Security:

* **Automation**: Automation is critical to supply chain security and can significantly reduce the possibility of human error and configuration drift.

* **Clarity**: The build environments used in a supply chain should be clearly defined, with limited scope.
Our upcoming blog post will cover the automation of all the steps discussed in this article, including container and metadata artifact signing, within our DevSecOps pipeline. Additionally, we’ll introduce other Open Source tools and projects, like Sigstore or Cosign, to further enhance the security of our Software Supply Chains.

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Stay tuned to the next blog post!

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>