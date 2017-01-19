---
layout:     post
title:      "Inside JD.COM's Shift To Kubernetes From OpenStack"
subtitle:   "Inside JD.COM's Shift To Kubernetes From OpenStack"
date:       2017-01-19 00:00:00
author:     "IPD"
header-img: "img/2017-01-19-inside-JD-shift-to-kubernetes-from-openstack/header.jpg"
catalog: true
tags:
    - kubernetes
---


# Intro

JD.COM (Jingdong) officially involved in the field of E-commerce in 2004. JD.COM begins trading on NASDAQ on May 22, 2014 under the ticker “JD”. In the following year, market transactions of JD.COM reached RMB 462.7 billion while net income reached RMB 181.3 billion. Now, JD.COM's revenue scale is the largest in Chinese Internet companies. And in July 2016, JD.COM became the first Chinese Internet company to make the Global Fortune 500 list.

# History of cluster building

### The era of physical machines (2004-2014)

Before 2014, the company's applications were all deployed in the physical machine. In the era of physical machines, we need to wait an average of one week for the allocation to application on-line. Due to the lack of isolation, applications would affected each other, resulting in a lot of potential risks. At that time, the average number of tomcat instances on each physical machine was no more than nine. The resource of physical machine was seriously wasted and the scheduling was inflexible. The time of application migration took hours due to the breakdown of physical machines. And the auto-scaling cannot be achieved. To enhance the efficiency of application deployment, the company has developed such as compilation-packaging, automatic deployment, log collection, resource monitoring and some other systems.

### Containerized era (2014-2016)

The Infrastructure Platform Department ([IPD](https://github.com/ipdcode)) led by Liu Haifeng--Chief Architect of JD.COM, sought a new resolution in the fall of 2014. Docker ran into our horizon. At that time, docker had been rising, but was slightly weak and lacked of experience in production environment. We had repeatedly tested docker. In addition, docker was customized to fix a couple of issues, such as system crash caused by device mapper and some Linux kernel bugs. We also added plenty of new features into docker, including disk speed limit, capacity management, and layer merging in image building and so on.

To manage the container cluster properly, we chose the architecture of OpenStack + Novadocker driver. Containers are managed as virtual machines. It is known as the first generation of JD container engine platform--JDOS1.0 (JD Datacenter Operating System). The main purpose of JDOS 1.0 is to containerize the infrastructure. All applications run in containers rather than physical machines since then. As for the operation and maintenance of applications, we took full advantage of existing tools. The time for developers to request computing resources in production environment reduced to several minutes rather than a week. After the pooling of computing resources, even the scaling of 1000 containers would be finished in seconds. Application instances had been isolated from each other. Both the average deployment density of applications and the physical machine utilization had increased by three times, which brought great economic benefits.  
We deployed clusters in each IDC and provided unified global APIs to support deployment across the IDC. There are 10,000 compute nodes at most and 4,000 at least in a single OpenStack distributed container cluster in our production environment. The first generation of container engine platform (JDOS 1.0) successfully supported the “6.18” and “11.11” promotional activities in both 2015 and 2016. There are already 150,000 running containers online by November 2016.

> “6.18” and “11.11”are known as the two most popular online promotion of JD.COM, similar to the black Friday promotions. Fulfilled orders in November 11, 2016 reached 30 million. 

In the practice of developing and promoting JDOS 1.0, applications were migrated directly from physical machines to containers. Essentially, JDOS 1.0 was an implementation of IaaS. Therefore, deployment of applications was still heavily dependent on compilation-packaging and automatic deployment tools. However, the practice of JDOS1.0 is very meaningful. Firstly, we successfully moved business into containers. Secondly, we have a deep understanding of container network and storage, and know how to polish them to the best. Finally, all the experiences lay a solid foundation for us to develop a brand new application container platform.

# New container engine platform (JDOS 2.0)

### platform architecture

When JDOS 1.0 grew from 2000 containers to 100,000, we launched a new container engine platform (JDOS 2.0). The goal of JDOS 2.0 is not just an infrastructure management platform, but also a container engine platform faced to applications. On the basic of JDOS 1.0 and kubernetes,  JDOS 2.0 integrates the storage and network of JDOS 1.0,  gets through the process of CI/CD from the source to the image, and finally to the deployment. Also, JDOS 2.0 provides one-stop service such as log, monitor, troubleshooting, terminal and orchestration. The platform architecture of JDOS 2.0 is shown below.

![arc](/img/2017-01-19-inside-JD-shift-to-kubernetes-from-openstack/arc.png)

| Function               | Product                   |
| ---------------------- | ------------------------- |
| Source Code Management | Gitlab                    |
| Container Tool         | Docker                    |
| Container Networking   | Cane                      |
| Container Engine       | kubernetes                |
| Image Registry         | Harbor                    |
| CI Tool                | Jenkins                   |
| Log Management         | Logstash + Elastic Search |
| Monitor                | Prometheus                |

In JDOS 2.0, we define two levels, system and application. A system consists of several applications and an application consists of several Pods which provide the same service. In general, a department can apply for one or more systems which directly corresponds to the namespace of kubernetes. This means that the Pods of the same system will be in the same namespace.

Most of the JDOS 2.0 components (Gitlab / Jenkins / Harbor / Logstash / Elastic Search / Prometheus) are also containerized and deployed on the kubernetes platform.

### one stop solution


JDOS 2.0 takes docker image as the core to implement continuous integration and continuous deployment.

![cicd](/img/2017-01-19-inside-JD-shift-to-kubernetes-from-openstack/cicd.png)

1. Developer pushes code to git.
2. Git triggers the jenkins master to generate build job.
3. Jenkins master invokes kubernetes to create jenkins slave Pod.
4. Jenkins slave pulls the source code, compiles and packs.
5. Jenkins slave sends the package and the Dockerfile to the image build node with docker.
6. The image build node builds the image.
7. The image build node pushes the image to the image registry Harbor.
8. User creates or updates app Pods in different zone.

The docker image in JDOS 1.0 consisted primarily of the operating system and the runtime software stack of the application. So, the deployment of applications was still dependent on the auto-deployment and some other tools. While in JDOS 2.0, the deployment of the application is done during the image building. And the image contains the complete software stack, including App. With the image, we can achieve the goal of running applications as designed in any environment.

![image](/img/2017-01-19-inside-JD-shift-to-kubernetes-from-openstack/image.png)

### networking and external service load balancing

JDOS 2.0 takes the network solution of JDOS 1.0, which is implemented with the VLAN model of OpenStack Neutron. This solution enables highly efficient communication between containers, making it ideal for a cluster environment within a company. Each Pod occupies a port in Neutron, with a separate IP. Based on the CNI standard, we have developed a new project Cane for integrating kubelet and Neutron.


![network](/img/2017-01-19-inside-JD-shift-to-kubernetes-from-openstack/network.png)

At the same time, Cane is also responsible for the management of LoadBalancer in kubernetes service. When a LoadBalancer is created / deleted / modified, Cane will call the creating / removing / modifying interface of the lbaas service in Neutron.  In addition, the Hades component in the Cane project provides an internal DNS resolution service for the Pods.

>  The source code of Cane project is currently in finishing. Later it will be open on github.

### flexible scheduling

JDOS 2.0 accesses applications, including big data, web applications, deep learning and some other types, and takes more diverse and flexible scheduling approaches. In some IDCs, we experimentally mixed deployment of online tasks and offline tasks. Compared to JDOS 1.0, overall resource utilization increased by about 30%.

![schedule](/img/2017-01-19-inside-JD-shift-to-kubernetes-from-openstack/schedule.png)

# Summary

The rich functionality of kubernetes allows us to pay more attention to the entire ecosystem of the platform, such as network performance, rather than the platform itself. In particular, the SREs highly appreciated the functionality of replication controller. With it, the scaling of the applications is achieved in several seconds. JDOS 2.0 now has accessed about 20% of the applications, and deployed 2 clusters with about 20,000 Pods running daily. We plan to access more applications of our comany, to replace the current JDOS 1.0. And we are also glad to share our experience in this process with the community.

Thanks to all the contributors of kubernetes and other open source projects.


