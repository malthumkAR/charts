<<<<<<< HEAD
# Kubeadm Ansible Playbook

Build a Kubernetes cluster using Ansible with kubeadm. The goal is easily install a Kubernetes cluster on machines running:

  - Ubuntu 16.04
  - CentOS 7
  - Debian 9

System requirements:

  - Deployment environment must have Ansible `2.4.0+`
  - Master and nodes must have passwordless SSH access

# Usage

Add the system information gathered above into a file called `hosts.ini`. For example:
```
[master]
192.16.35.12

[node]
192.16.35.[10:11]

[kube-cluster:children]
master
node
```

If you're working with ubuntu, add the following properties to each host `ansible_python_interpreter='python3'`:
```
[master]
192.16.35.12 ansible_python_interpreter='python3'

[node]
192.16.35.[10:11] ansible_python_interpreter='python3'

[kube-cluster:children]
master
node

```

Before continuing, edit `group_vars/all.yml` to your specified configuration.

For example, I choose to run `flannel` instead of calico, and thus:

```yaml
# Network implementation('flannel', 'calico')
network: flannel
```

**Note:** Depending on your setup, you may need to modify `cni_opts` to an available network interface. By default, `kubeadm-ansible` uses `eth1`. Your default interface may be `eth0`.

After going through the setup, run the `site.yaml` playbook:

```sh
$ ansible-playbook site.yaml
...
==> master1: TASK [addon : Create Kubernetes dashboard deployment] **************************
==> master1: changed: [192.16.35.12 -> 192.16.35.12]
==> master1:
==> master1: PLAY RECAP *********************************************************************
==> master1: 192.16.35.10               : ok=18   changed=14   unreachable=0    failed=0
==> master1: 192.16.35.11               : ok=18   changed=14   unreachable=0    failed=0
==> master1: 192.16.35.12               : ok=34   changed=29   unreachable=0    failed=0
```

The playbook will download `/etc/kubernetes/admin.conf` file to `$HOME/admin.conf`.

If it doesn't work download the `admin.conf` from the master node:

```sh
$ scp k8s@k8s-master:/etc/kubernetes/admin.conf .
```

Verify cluster is fully running using kubectl:

```sh

$ export KUBECONFIG=~/admin.conf
$ kubectl get node
NAME      STATUS    AGE       VERSION
master1   Ready     22m       v1.6.3
node1     Ready     20m       v1.6.3
node2     Ready     20m       v1.6.3

$ kubectl get po -n kube-system
NAME                                    READY     STATUS    RESTARTS   AGE
etcd-master1                            1/1       Running   0          23m
...
```

# Resetting the environment

Finally, reset all kubeadm installed state using `reset-site.yaml` playbook:

```sh
$ ansible-playbook reset-site.yaml
```

# Additional features
These are features that you could want to install to make your life easier.

Enable/disable these features in `group_vars/all.yml` (all disabled by default):
```
# Additional feature to install
additional_features:
  helm: false
  metallb: false
  healthcheck: false
```

## Helm
This will install helm in your cluster (https://helm.sh/) so you can deploy charts.

## MetalLB
This will install MetalLB (https://metallb.universe.tf/), very useful if you deploy the cluster locally and you need a load balancer to access the services.

## Healthcheck
This will install k8s-healthcheck (https://github.com/emrekenci/k8s-healthcheck), a small application to report cluster status.

# Utils
Collection of scripts/utilities

## Vagrantfile
This Vagrantfile is taken from https://github.com/ecomm-integration-ballerina/kubernetes-cluster and slightly modified to copy ssh keys inside the cluster (install https://github.com/dotless-de/vagrant-vbguest is highly recommended)

# Tips & Tricks
## Specify user for Ansible
If you use vagrant or your remote user is root, add this to `hosts.ini`
```
[master]
192.16.35.12 ansible_user='root'

[node]
192.16.35.[10:11] ansible_user='root'
```

## Access Kubernetes Dashboard
As of release 1.7 Dashboard no longer has full admin privileges granted by default, so you need to create a token to access the resources:
```sh
$ kubectl -n kube-system create sa dashboard
$ kubectl create clusterrolebinding dashboard --clusterrole cluster-admin --serviceaccount=kube-system:dashboard
$ kubectl -n kube-system get sa dashboard -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2017-11-27T17:06:41Z
  name: dashboard
  namespace: kube-system
  resourceVersion: "69076"
  selfLink: /api/v1/namespaces/kube-system/serviceaccounts/dashboard
  uid: 56b880bf-d395-11e7-9528-448a5ba4bd34
secrets:
- name: dashboard-token-vg52j

$ kubectl -n kube-system describe secrets dashboard-token-vg52j
...
token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtdG9rZW4tdmc1MmoiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNTZiODgwYmYtZDM5NS0xMWU3LTk1MjgtNDQ4YTViYTRiZDM0Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZCJ9.bVRECfNS4NDmWAFWxGbAi1n9SfQ-TMNafPtF70pbp9Kun9RbC3BNR5NjTEuKjwt8nqZ6k3r09UKJ4dpo2lHtr2RTNAfEsoEGtoMlW8X9lg70ccPB0M1KJiz3c7-gpDUaQRIMNwz42db7Q1dN7HLieD6I4lFsHgk9NPUIVKqJ0p6PNTp99pBwvpvnKX72NIiIvgRwC2cnFr3R6WdUEsuVfuWGdF-jXyc6lS7_kOiXp2yh6Ym_YYIr3SsjYK7XUIPHrBqWjF-KXO_AL3J8J_UebtWSGomYvuXXbbAUefbOK4qopqQ6FzRXQs00KrKa8sfqrKMm_x71Kyqq6RbFECsHPA

$ kubectl proxy
```
> Copy and paste the `token` from above to dashboard.

Login the dashboard:
- Dashboard: [https://API_SERVER:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/](https://API_SERVER:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/)
- Logging: [https://API_SERVER:8001/api/v1/namespaces/kube-system/services/kibana-logging/proxy/](https://API_SERVER:8001/api/v1/namespaces/kube-system/services/kibana-logging/proxy/)

=======
# Helm Charts

The canonical source for Helm charts is the [Helm Hub](https://hub.helm.sh/), an aggregator for distributed chart repos.

This GitHub project is the source for Helm `stable` and `incubator` [Helm chart repositories](https://v3.helm.sh/docs/topics/chart_repository/), currently listed on the Hub.

For more information about installing and using Helm, see the
[Helm Docs](https://helm.sh/docs/). For a quick introduction to Charts, see the [Chart Guide](https://helm.sh/docs/topics/charts/).

## Status of the Project

Similar to the [Helm 2 Support Plan](https://helm.sh/blog/2019-10-22-helm-2150-released/#helm-2-support-plan), this GitHub project has begun transition to a 1 year "maintenance mode" (see [Deprecation Timeline](#deprecation-timeline) below). Given the deprecation plan, this project is intended for [apiVersion: v1](https://helm.sh/docs/topics/charts/#the-apiversion-field) Charts (installable by both Helm 2 and 3), and not for `apiVersion: v2` charts (installable by Helm 3 only).

### Deprecation Timeline

| | |
| - | - |
| **Nov 13, 2019** | At [Helm 3's public release](https://helm.sh/blog/helm-3-released/), new charts are no longer accepted to `stable` or `incubator`. Patches to existing charts may continue to be submitted by the community, and (time permitting) reviewed by chart OWNERS for acceptance |
| **Aug 13, 2020** | At 9 months – when Helm v2 goes security fix only – the `stable` and `incubator` repos will be de-listed from the Helm Hub. Chart OWNERS are encouraged to accept security fixes only. ℹ️ _Note: the original date has been extended 3 months to match Helm v2 support. See [COVID-19: Extending Helm v2 Bug Fixes](https://helm.sh/blog/covid-19-extending-helm-v2-bug-fixes/)_ |
| **Nov 13, 2020** | At 1 year, support for this project will formally end, and this repo will be marked obsolete |

This timeline gives the community (chart OWNERS, organizations, groups or individuals who want to host charts) 9 months to move charts to new Helm repos, and list these new repos on the Helm Hub before `stable` and `incubator` are de-listed.

Note that this project has been under active development for some time, so you might run into [issues](https://github.com/helm/charts/issues). If you do, please don't be shy about letting us know, or better yet, contribute a fix or feature (within the deprecation timeline of course). Also be aware the repo and chart OWNERS are volunteers so reviews are as time allows, and acceptance is up to the chart OWNERS - you may [reach out](#where-to-find-us) but please be patient and courteous.

## Where to Find Us

For general Helm Chart discussions join the Helm Charts (#charts) room in the [Kubernetes Slack instance](http://slack.kubernetes.io/).

For issues and support for Helm and Charts see [Support Channels](CONTRIBUTING.md#support-channels).

## How Do I Install These Charts?

Just `helm install stable/<chart>`. This is the default repository for Helm which is located at https://kubernetes-charts.storage.googleapis.com/ and is installed by default.

For more information on using Helm, refer to the [Helm documentation](https://github.com/kubernetes/helm#docs).

## How Do I Enable the Stable Repository for Helm 3?

To add the Helm Stable Charts for your local client, run `helm repo add`:

```
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com
"stable" has been added to your repositories
```

## How Do I Enable the Incubator Repository?

To add the Incubator charts for your local client, run `helm repo add`:

```
$ helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com
"incubator" has been added to your repositories
```

You can then run `helm search incubator` to see the charts.

## Chart Format

Take a look at the [alpine example chart](https://github.com/helm/helm/tree/master/cmd/helm/testdata/testcharts/alpine) for reference when you're writing your first few charts.

Before contributing a Chart, become familiar with the format. Note that the project is still under active development and the format may still evolve a bit.

## Repository Structure

This GitHub repository contains the source for the packaged and versioned charts released in the [`gs://kubernetes-charts` Google Storage bucket](https://console.cloud.google.com/storage/browser/kubernetes-charts/) (the Chart Repository).

The Charts in the `stable/` directory in the master branch of this repository match the latest packaged Chart in the Chart Repository, though there may be previous versions of a Chart available in that Chart Repository.

The purpose of this repository is to provide a place for maintaining and contributing official Charts, with CI processes in place for managing the releasing of Charts into the Chart Repository.

The Charts in this repository are organized into two folders:

* stable
* incubator

Stable Charts meet the criteria in the [technical requirements](CONTRIBUTING.md#technical-requirements).

Incubator Charts are those that do not meet these criteria. Having the incubator folder allows charts to be shared and improved on until they are ready to be moved into the stable folder. The charts in the `incubator/` directory can be found in the [`gs://kubernetes-charts-incubator` Google Storage Bucket](https://console.cloud.google.com/storage/browser/kubernetes-charts-incubator).

In order to get a Chart from incubator to stable, Chart maintainers should open a pull request that moves the chart folder.

## Contributing to an Existing Chart

We'd love for you to contribute to an existing Chart that you find provides a useful application or service for Kubernetes. Please read our [Contribution Guide](CONTRIBUTING.md) for more information on how you can contribute Charts.

Note: We use the same [workflow](https://github.com/kubernetes/community/blob/master/contributors/devel/development.md#workflow),
[License](LICENSE) and [Contributor License Agreement](CONTRIBUTING.md) as the main Kubernetes repository.

## Owning and Maintaining A Chart

Individual charts can be maintained by one or more users of GitHub. When someone maintains a chart they have the access to merge changes to that chart. To have merge access to a chart someone needs to:

1. Be listed on the chart, in the `Chart.yaml` file, as a maintainer. If you need sponsors and have contributed to the chart, please reach out to the existing maintainers, or if you are having trouble connecting with them, please reach out to one of the [OWNERS](OWNERS) of the charts repository.
1. Be invited (and accept your invite) as a read-only collaborator on [this repo](https://github.com/helm/charts). This is required for @k8s-ci-robot [PR comment interaction](https://github.com/kubernetes/community/blob/master/contributors/guide/pull-requests.md).
1. An OWNERS file needs to be added to a chart. That OWNERS file should list the maintainers' GitHub login names for both the reviewers and approvers sections. For an example see the [Drupal chart](stable/drupal/OWNERS). The `OWNERS` file should also be appended to the `.helmignore` file.

Once these three steps are done a chart approver can merge pull requests following the directions in the [REVIEW_GUIDELINES.md](REVIEW_GUIDELINES.md) file.

## Trusted Collaborator

The `pull-charts-e2e` test run, that installs a chart to test it, is required before a pull request can be merged. These tests run automatically for members of the Helm Org and for chart [repository collaborators](https://help.github.com/articles/adding-outside-collaborators-to-repositories-in-your-organization/). For regular contributors who are trusted, in a manner similar to Kubernetes community members, we have trusted collaborators. These individuals can have their tests run automatically as well as mark other pull requests as ok to test by adding a comment of `/ok-to-test` on pull requests.

There are two paths to becoming a trusted collaborator. One only needs follow one of them.

1. If you are a Kubernetes GitHub org member and have your Kubernetes org membership public you can become a trusted collaborator for Helm Charts
2. Get sponsorship from one of the Charts Maintainers listed in the OWNERS file at the root of this repository

The process to get added is:

* File an issue asking to be a trusted collaborator
* A Helm Chart Maintainer can then add the user as a read only collaborator to the repository

## Review Process

For information related to the review procedure used by the Chart repository maintainers, see [Merge approval and release process](CONTRIBUTING.md#merge-approval-and-release-process).

### Stale Pull Requests and Issues

Pull Requests and Issues that have no activity for 30 days automatically become stale. After 30 days of being stale, without activity, they become rotten. Pull Requests and Issues can rot for 30 days and then they are automatically closed. This is the standard stale process handling for all repositories on the Kubernetes GitHub organization.

## Supported Kubernetes Versions

This chart repository supports the latest and previous minor versions of Kubernetes. For example, if the latest minor release of Kubernetes is 1.8 then 1.7 and 1.8 are supported. Charts may still work on previous versions of Kubernertes even though they are outside the target supported window.

To provide that support the API versions of objects should be those that work for both the latest minor release and the previous one.

## Happy Helming in China

If you are in China, there are some problems to use upstream Helm Charts directly (e.g. images hosted on `gcr.io`, `quay.io`, and Charts hosted on `googleapis.com` etc), you can use this mirror repo at https://github.com/cloudnativeapp/charts which automatically sync & replace unavailable image & repo URLs in every Chart.
>>>>>>> 6ed964a07209f4614c8cd8a952cec83305a79b7f
