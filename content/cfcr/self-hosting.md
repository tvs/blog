---
title: "Self-Hosting CFCR"
date: 2018-06-05
tags: ["cfcr", "architecture", "self-hosting"]
draft: false
---
# Self-Hosting CFCR

## What is Self-Hosting?

  "Self-hosting" is a concept that is ubiquitous in many different categories of
computing. At its core, "self-hosting" is simply the ability for a system to be
expressed by itself. One of the most common examples is in the world of
programming languages where the compiler is often written in the very same
language that the compiler is written _for_. Another common scenario can be seen
with kernels where developers use older versions of the kernel to work on newer
ones. Often times self-hosting requires an initial implementation that can be
used to get the system into a steady self-hosting capable state - sometimes done
through cross-compilation - through a technique called bootstrapping. Once
the initial bootstrapping has been done, the self-hosted system should be
readily available to begin expressing new versions of itself.

  In the context of Kubernetes, self-hosting means running the entire Kubernetes
system on the cluster as though it were just another Kubernetes workload. That
means `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `kubelet`,
and the other components which make up the Kubernetes control plane are deployed
using Kubernetes objects (e.g. `Pods`, `DaemonSets`, and `Deployments`) and
subject to management by the very control plane that they compose.

  Self-hosting Kubernetes was first, officially, proposed [back in 2014](https://github.com/kubernetes/kubernetes/issues/246#issuecomment-64533959) and has
seen [continued](https://github.com/kubernetes/kubernetes/issues/4090#issuecomment-74890508) [development](https://github.com/kubernetes-incubator/bootkube) [since then](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/cluster-lifecycle/self-hosted-kubernetes.md). During the initial proposal, Brian Grant
suggested a simple recipe:

  1. Break circular dependencies by converting hard dependencies to soft
     dependencies.
    * Make all state rediscoverable or reconstructable.
    * Allow pivoting from bootstrap instances to steady state.
    * Accept data from other components and sources, such as local files, that
      can be used at bootstrap time.
    * Restart components such as `kubelet`, which run locally.
  3. Minimize the number of dependencies required for steady-state operation.
  4. Stratify the dependencies that remain via pricipled layering.

  Since then, the concept of "self-hosting" has been classified into a spectrum
described in this graphic:

![Self-Hosted Layers](/img/self-hosted-layers.png)

  In this hierarchy, the Kubernetes components are layered starting from the
most basic component, the `kubelet` at layer zero, and progressively works its
way up the component dependency list until it reaches the optional components
like `kube-dns` and other cluster addons at layer four. In this way, a cluster
can be referenced by the layers that it self-hosts. A 0-4 self-hosted cluster
would reference a cluster where `kubelet` is deployed to the various VMs as a
`DaemonSet`, `etcd` is running as a workload in the cluster, `kube-apiserver`
runs as a `Pod` and is exposed as a `Service`, and the `kube-scheduler`,
`kube-controller-manager`, and `kube-proxy` are running as their own respective
workloads. In a 1-4 self-hosted cluster, the VMs run their own system-installed
`kubelet`, and a 2-4 cluster would have `etcd` running outside of Kubernetes.
Since Kubernetes 1.4, [Bootkube](https://github.com/kubernetes-incubator/bootkube) and [Tectonic](https://coreos.com/tectonic/) have been providing 1-4 and 2-4
self-hosted clusters and more recently, Bootkube has also begun providing 0-4
self-hosted clusters as an alpha feature.

  It should be noted that self-hosting was never intended to _replace_ or
_impede_ other methods for installing and configuring Kubernetes. Rather it's
intended to exist alongside those methods and come with its own advantages and
disadvantages. [The official community proposal for a self-hosted control plane](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/cluster-lifecycle/self-hosted-kubernetes.md)
lists the following advantages:

>  1. *Small Dependencies:* self-hosted should reduce the number of components
>    required, on host, for a Kubernetes cluster to be deployed to a Kubelet
>    (ideally running in a container). This should greatly simply the perceived
>    complexity of Kubernetes installation.
>  2. *Deployment Consistency:* self-hosted reduces the number of files that are
>    written to disk or managed via configuration management or manual
>    installation via SSH. Our hope is to reduce the number of moving parts
>    relying on the host OS to make deployments consistent in all environments.
>  3. *Introspection:* internal components can be debugged and inspected by
>     users using existing Kubernetes APIs like `kubectl logs`.
>  4. *Cluster Upgrades:* Related to introspection the components of a
>     Kubernetes cluster are now subject to control via Kubernetes APIs.
>     Upgrades of Kubelet's are possible via new daemon sets, API servers can
>     be upgraded using daemon sets and potentially deployments in the future,
>     and flags of add-ons can be changed by updating deployments, etc.
>  5. *Easier Highly-Available Configurations:* Using Kubernetes APIs will make
>     it easier to scale up and monitor an HA environment without complex
>     external tooling. Because of the complexity of these configuration tools
>     that create them without self-hosted often implement significant complex
>     logic.

  Some of these concepts should sound familiar to CFCR-ians:
*Deployment Consistency*, *Cluster Upgrades* and *Easier Highly-Available Configurations*
are traditionally the purview of BOSH, and so the question that follows is
"what does self-hosting bring to CFCR?"

## What Does Self-Hosting Bring to CFCR?

  CFCR is a strange beast, in that most of what the project aims to do is
repackage and distribute other software. Software that is fundamentally very
similar to the mechanism through which it is being deployed (BOSH). From that
angle, the CFCR development team interacts with it in a manner very similar to
its customers.  The team often finds itself delving into the logs of components
like `kube-proxy` or `kube-controller-manager` to figure out why certain
features are not behaving.  Application developers can often find themselves in
the same situation, reading through logs on `kube-apiserver` or the `kubelet`
to figure out why their application fails to deploy, why their networking is
failing, or why their deployment is not scaling. While CFCR ideally aims to
minimize the need for developers to concern themselves with those types or
problems, the reality is that there is only so much that can be done -
Kubernetes is a monolith of complexity and the open-endedness of what can be
deployed through it makes it impossible to prevent all such problems.

  Because of the similarities between the team and customers, the values that
self-hosting brings to CFCR are hard to place into strict "product" or
"technical" categories. Many of the technical benefits are also product
benefits and many of the product benefits can help with paying down technical
debt, improving product maintenance, and bringing the team closer to a unified
set of deliverables. With that preface, the benefits are thus:

  1. *Introspection:* the experience of managing Kubernetes will become
     significantly more unified. Operators will be able to debug their cluster
     using the same mechanisms, tools, and techniques that they use to debug
     workloads. `kubectl logs` becomes the primary entry point for as many
     components as we can self-host.
  2. *Easier HA Configuration:* the API components become as manageable as any
     other workload. The manifests for components like `kube-apiserver` can be
     modified live in the cluster without needing to involve BOSH, the control
     plane components can be referenced with other API objects like load
     balancers, we can scale them up or down independent of the master VMs (to
     assist in surge scheduling, for example), and we can hook them into other
     cluster workloads (e.g. monitoring amd metrics).
  3. *Kubelet on the Master:* by putting a `kubelet` on the Master VMs, we can
     readily deploy workloads to them _without_ having to go through the
     complications of building a BOSH release. That means cluster administrators
     can deploy monitoring and metrics tools that keep track of API load,
     monitor access logs for suspicious activity, and readily schedule backups
     for master components (e.g. `etcd`). In essence, the Master VM becomes
     very similar to a Worker VM, only the `kubelet` is deployed with a special
     set of labels denoting that it's only intended for a specific type of
     workload (e.g. the control plane components).
  4. *Minimizes Components*: with 1-4 self-hosting, CFCR would essentially only
     need to manage a few things: `kubelet`, `docker`, and a set of bootstrapping
     instructions that are used to bring up a temporary control plane. The other
     components would become a series of manifests (and Docker images for
     airgapped environments) that get deployed to the bootstrapped cluster and
     later maintained through Kubernetes. That would mean we would no longer
     have to concern ourseles with packaging and maintaining binaries for the
     control plane, we would no longer have to worry about installing software
     required for those components (e.g. `conntrack` or `ipset` for `kube-proxy`)
     as the container images already include them, and any special sauce that
     CFCR or CFCR-based products wish to deploy can be kept in isolation from
     one another.
  5. *API Configuration:* because API components are yet-another-workload it's
     possible for cluster administrators to change their configuration _after_
     deployment by rolling out changes to their manifest. This gives the
     operator a much wider range of flexibility when it comes to what features
     their cluster enables or disables and removes some of the burden from CFCR
     to constantly track new features in the job specs.
  6. *Interchangeable Parts:* the cluster operator can leverage the powerful
     `kube-aggregator` [to deploy their own custom API servers](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/aggregated-api-servers.md), allowing
     CFCR-based products to supply their own flavors of components.
     Additionally, the operator is no longer tied to `flannel` and can use the
     CNI-recommended Kubernetes manifests to deploy one of their choosing
     instead of having to wrap it all up in a complicated BOSH release. Further
     `etcd` can be managed in more sophisticated ways through
     Kubernetes-centric tools like `etcd-operator`, and the
     `cloud-controller-manager` can be realized as a properly configurable
     mechanism for leveraging out-of-tree cloud providers.
  7. *Sophisticated etcd Management:* `etcd` is the constant weakpoint of any
     Kubernetes cluster and the CFCR team simply does not have the bandwidth,
     experience, expertise, or drive to pursue complex solutions to the problems
     surrounding its management. Fortunately, CoreOS _does_ and has been working
     on tools like `etcd-operator` for managing `etcd` clusters within
     Kubernetes.  Self-hosting `etcd` would allow us to leverage these tools
     instead of providing our own, frankly, bare-bones solutions.
  8. *Backup and Restore is Unified:* there would be no need for multiple backup
     and restore solutions. `etcd` could be treated the same way as any other
     workload. While much of the story around Kubernetes backup and restore is
     in its infancy (e.g. [Heptio's Ark](https://github.com/heptio/ark), [Storage Volume Snapshots](https://github.com/kubernetes/features/issues/177)
     and [Volume Snapshot Controller](https://github.com/kubernetes-incubator/external-storage/tree/master/snapshot)), there is value to be had from unifying
     that solution rather than pursuing necessarily disparate ones with the
     control plane being a BOSH component and the workloads existing within
     Kubernetes.
  9. *Self-Regulation of API Components:* API components can utilize their own
     `ServiceAccount` rather than the cludgy users defined in a token file,
     they can define their own `Roles`, `Secrets`, `ConfigMaps`, or other
     objects related to their operation, and those can be placed directly within
     their manifest, keeping everything within close relation to one another.
     This means there would no longer be confusing circular dependencies between
     where a job is run and where the `ClusterRole` and `ClusterRoleBindings`
     are defined. In short, there would be no need for the `kubernetes-roles`
     job that is so tightly coupled with the proper operation of certain
     components.
  10. *Comparable Architecture:* this point is sort of a stretch, but it bears
      mentioning. By modeling the architecture closer to a community standard
      that is being heavily leveraged by other community-provided tooling (e.g.
      Bootkube, [kops](https://github.com/kubernetes/kops), [kubeadm](https://github.com/kubernetes/kubeadm/blob/master/docs/design/design_v1.10.md#optional-and-alpha-in-v19-self-hosting), Tectonic, and
      [a number of other Bootkube-based solutions](https://github.com/kubernetes-incubator/bootkube/blob/master/Documentation/users-integrations.md)) it becomes far more likely that
      the problems CFCR encounters have already been seen, reported and
      sometimes solved. When those problems still need a solution, it also will
      be much easier to get buy-in, help, and guidance when it comes to fixing
      them, because CFCR is not in some unique scenario. From the past, CFCR
      has had issues with `hostname-override` that we have brought to the
      Kubernetes community and the efforts around fixing them have been
      rebuffed. If CFCR had a more standardized architecture (one more heavily
      divorced from the limitations of BOSH), it becomes much less likely that
      the problems would be seen as a one-off issue with a relatively unknown
      deployment tool.
  11. *Unified Logging Solutions:* today, advanced logging options are handled
      disparately for API components and workloads. BOSH handles log rotation
      for components like `kube-apiserver` and `kubelet` while it's left as an
      exercise for the operator on workloads. With a self-hosted cluster we can
      both leverage and expose addons that handle log rotation and backup for
      _all_ components. Of course, operators that prefer custom solutions would
      have the flexibility to set that up as well. Products like [fluentd](https://www.fluentd.org/) are
      already commonly used alongside [Google Stackdriver](https://cloud.google.com/stackdriver/) in the GKE ecosystem.

  Of course, self-hosting is not all sunshine and puppy dogs. While there are
many benefits, there are significant challenges associated with it too.

## Challenges in Self-Hosting CFCR

  At this point, any sufficiently skeptical person should be asking the age old
question, "what's the catch?" And of course, they'd be right to do so. There are
a few challenges that revolve around the process of self-hosting as well as
problems created _because_ of self-hosting.

  1. *Breaking BOSH Contracts:* in the BOSH world, it's expected that the jobs
     are bundled together through the release and it's through the release object
     that they can be managed and upgraded. With a self-hosted Kubernetes cluster
     it's possible for the operator to leverage Kubernetes to roll out upgrades
     of itself independent from the version of `kubo-release`. The release can
     be configured so that it doesn't override existing deployment manifests, but
     that sort of blurs the line of what CFCR actually is.
  2. *Blurring the Line:* if `kubo-release` isn't the only mechanism for rolling
     out upgrades of Kubernetes components, CFCR's identity becomes a little
     less clear. Is CFCR only a bootstrapping tool for bringing up a Kubernetes
     cluster on BOSH? Does it provide an official, _supported_, mechanism for
     rolling out upgrades to those components and leaves other CFCR-based projects
     to figure out their own upgrades? CFCR could readily become a framework for
     building flavors of Kubernetes cluster (e.g. PKS) while also providing its
     own vanilla variant, but it would require a much clearer definition of self
     than it currently has.
  3. *Upgrades:* moving from the existing architecture to a self-hosted cluster
     might be difficult because of `etcd`. How do we transfer data from the
     existing, out-of-cluster `etcd` into the in-cluster `etcd`? Assuming we
     choose to support out-of-cluster `etcd` (which is another topic to be
     discussed), how do we transition the data out of our cluster? The logistics
     of adding another node to the `etcd` cluster mid-upgrade might be cumbersome
     with the way BOSH operates.
  4. *Tenancy Problems:* once a Kubelet exists on the Master VM, the tenancy
     problems that have plagued PKS become much more difficult to avoid. While
     CFCR will set the `labels` and `taints` that generally serve to prevent
     scheduling workloads on the Master nodes, it's possible to set `tolerations`
     that will schedule to them anyway. In general, this is more a problem with
     tenancy in Kubernetes (which only offers "soft tenancy") and the way it's
     being used in downstream products, but it can be a hurdle that would need
     to be solved. Arguably, CFCR should not need to concern itself with this
     issue, so long as we also provide the flexibility required for those
     downstream products to solve it (via `AppArmor`, `PodSecurityPolicies`,
     custom admission controllers, or custom API servers, for example), but we
     would likely want to provide guidance on doing so.
  5. *Privileged Containers for CNI:* it's possible this doesn't hold true for
     _all_ of the CNI variants, but with `flannel`, it's necessary to run the
     container in a privileged state. Normally `allow-privileged` is something
     we've allowed to be controlled, but in a self-hosted scenario, it would
     become much more mandatory. The risks can be mitigated through other
     mechanisms like `PodSecurityPolicies` that prevent users other than those
     with the `cluster-admin` role (or CNI-specific role) from deploying said
     container, but the tenancy issues described elsewhere means we would have
     fairly significant downstream impact.

## Should CFCR Become Self-Hosted?

  While I'm not claiming that the list of pros or cons are exhaustive, I do
believe that they cover the biggest issues for each category. From that
perspectivel, I do believe that CFCR should aim to become self-hosted in the
future. Many of the benefits are ones which empower product users, product
developers, and downstream consumers. The product will become less dogmatic over
what is and is not possible, it becomes slimmer and thereby easier to maintain,
and it concentrates itself more around Kubernetes (which is the end product)
rather than BOSH. From the other end, the complications and drawbacks seem to be
ones that CFCR already struggles with: _what, exactly, is_ CFCR? And _how_ should
CFCR be used? While self-hosting may not necessarily _help_ in answering those
questions, it does little to harm them. At worst, it seems to make the answers
more pressing and immediate.

## Assorted Interesting Links

  * [CoreOS Blog: Self-Hosted Kubernetes makes Kubernetes installs, scaleouts, upgrades easier](https://coreos.com/blog/self-hosted-kubernetes.html)
    * [YouTube: Self-Hosted Kubernetes](https://www.youtube.com/watch?v=tXyV3IQ8-0k)
  * [YouTube: KubeCeption! A Story of Self-Hosted Kubernetes by Aaron Levy, CoreOS, Inc.](https://www.youtube.com/watch?v=EbNxGK9MwN4)
