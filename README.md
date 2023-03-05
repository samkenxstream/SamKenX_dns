# Kubernetes DNS

[![Build Status](https://travis-ci.org/kubernetes/dns.svg?branch=master)](https://travis-ci.org/kubernetes/dns)
[![Coverage Status](https://coveralls.io/repos/github/kubernetes/dns/badge.svg?branch=master)](https://coveralls.io/github/kubernetes/dns?branch=master)
[![Go Report Card](https://goreportcard.com/badge/github.com/kubernetes/dns)](https://goreportcard.com/report/github.com/kubernetes/dns)

This is the repository for [Kubernetes DNS](http://kubernetes.io/docs/admin/dns/).

## Images

* [kube-dns](http://kubernetes.io/docs/admin/dns/)
* [sidecar](docs/sidecar/README.md)
* [dnsmasq](images/dnsmasq)
* [node-cache](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/)

## Building

`make` targets:

| target | description |
| ---- | ---- |
|all, build   | build all binaries |
|test         | run unit tests |
|containers   | build the containers |
|images-clean | clear image build artifacts from workdir |
|push         | push containers to the registry |
|help         | this help message |
|version      | show package version |
|{build,containers,push}-ARCH | do action for specific ARCH |
|all-{build,containers,push}  | do action for all ARCH |
|only-push-BINARY             | push just BINARY |

* Setting `VERBOSE=1` will show additional build logging.
* Setting `VERSION` will override the container version tag.

## Vulnerability patching

Vulnerability patches are mainly for debian-base or debian-iptables images. They can be updated to the latest by modifying [rules.mk](https://github.com/kubernetes/dns/blob/master/rules.mk#L32-L33) and [dnsmasq Makefile](https://github.com/kubernetes/dns/blob/f44ede5f559a9a29fa23b438e6ce0cb70934d834/images/dnsmasq/Makefile#L30-L32).
[Example PR](https://github.com/kubernetes/dns/pull/475).

Once the PR has merged, a new release tag should be cut. The rest of the release process is described below.

## Release process
Follow these steps to make changes and release a new binary.

1. Make the necessary code changes and create a PR.
2. Build and test locally (`make images-clean`; `make build`; `make containers`; `make test`). 
3. To build just the node-cache container, use `make containers CONTAINER_BINARIES=node-cache`.
4. The same steps are executed via the presubmit script `presubmits.sh` which is run by the [test-infra prow job.](https://github.com/kubernetes/test-infra/blob/88cd2798f36010e071a30c9827f90e647b59fc65/config/jobs/kubernetes/sig-network/sig-network-misc.yaml#L182)
5. Merge the PR.
6. Cut a new release tag. We use [semantic versioning](http://semver.org) to
   name releases.
   Example:
   ```
   git tag -a 1.21.4 -m "Build images using golang 1.17."
   git push upstream 1.21.4
   ```
4. Wait for container images to be pushed via cloudbuild yaml. This will be done automatically by
   `k8s.io/test-infra/.../k8s-staging-dns.yaml`. A manual cloud build can be submitted via
   `gcloud builds submit --config cloudbuild.yaml`, but this requires owner permissions in k8s-staging-dns project.
   The automated job pushes images for all architectures and makes them available in `gcr.io/k8s-staging-dns`.
   Status for build jobs can be checked at - https://k8s-testgrid.appspot.com/sig-network-dns#dns-push-images
5. Promote the images to `gcr.io/k8s-artifacts-prod` using the process described
   in [this](https://github.com/kubernetes/k8s.io/tree/main/k8s.gcr.io#image-promoter) link.
   The image SHAs should be added to [`images/k8s-staging-dns/images.yaml`](https://github.com/kubernetes/k8s.io/blob/main/k8s.gcr.io/images/k8s-staging-dns/images.yaml).
   The SHAs can be obtained by running the command `python parse-image-sha.py <TAG>`
   This will return the SHAs for kube-dns as well as node-cache images. Node-cache images are always promoted, kube-dns images are promoted if there is a change to kubedns/vulnerability fix.
6. Images will be available in the repo registry.k8s.io/dns/. The node-cache image with tag 1.15.14 can be found at registry.k8s.io/dns/k8s-dns-node-cache:1.15.14. Older versions are at registry.k8s.io/k8s-dns-node-cache:<TAG>
7. Submit a PR for the kubernetes/kubernetes repository to switch to the new
   version of the containers. Example - https://github.com/kubernetes/kubernetes/pull/106189
