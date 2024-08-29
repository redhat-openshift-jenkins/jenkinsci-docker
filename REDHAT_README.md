# Red Hat Build of Jenkins Controller Image

This is a fork of the upstream [jenkinsci/docker](https://github.com/jenkinsci/docker) repository
which includes a separate Dockerfile for assembling a "clean" Jenkins image (located in
`rhel/ubi9/konflux`). This image can be run with Podman or any other container engine using the
same instructions in the main [README](./README.md). It does not contain any plugins for running
Jenkins pipelines or adding integrations for OpenShift.

## Branch Configuration

This fork should be align with upstream Jenkins to the furthest extent possible. The default
`master` branch should be kept in sync with the upstream repository.
Red Hat releases should use the branch convention `rh-jenkins-<version>`, where `<version>` is the
major and minor semantic version of the core Jenkins .war that is packaged. For example, the `2.462.z` LTS releases should use the branch name `rh-jenkins-2.462`.

## Onboarding to Konflux

The Dockerfile for building on [Konflux](https://konflux-ci.dev) should be kept in the
`rhel/ubi<N>/konflux` directory, where `<N>` is the major version of UBI used to assemble the
image. For example, `ubi9` base images should use the directory `rhel/ubi9/konflux` to place the
Dockerfile.

The Dockerfile for Konflux has several notable differences:

- We use our fork of [go-init](https://github.com/redhat-openshift-jenkins/go-init) as the main
  entrypoint. We have used this as the entrypoint for OpenShift's Jenkins since 2019, and it has
  proved itself stable in production. Upstream uses [tini](https://github.com/krallin/tini), which
  is C-based and provides almost identical functionality. `go-init` must be onboarded to Konflux
  and marked as a dependency. The binary is extracted from the component's container image.
- The core Jenkins .war file is extracted from the respective Konflux image. The `jenkins.war` file
  is likewise extracted from the component's container image, and must be marked as a dependency in
  Konflux.
- Ditto for the plugin manager .jar - it is extracted from the respective Konflux image.
- Rather than use standard UBI as the base (with some hacking to slim down the JDK), the Konflux
  Dockerfile uses standard "runtime" images published by Red Hat's OpenJDK maintainers. Notable
  differences are:
    - Images are currently `ubi-micro` based. For ubi9, `microdnf` is used to install RHEL
      packages.
    - `JAVA_HOME` is pre-set in the image, we do not provide our own "slim" JDK.
    - OpenJDK images create their own "default" user, which is separate from the "jenkins" user the
      image defaults to. Currently the UID for `default` (185) and `jenkins` (1000) do not collide.
      File permissions and the `HOME` environment variable are set accordingly so Jenkins runs in a
      secure manner by default.

