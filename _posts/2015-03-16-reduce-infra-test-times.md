---
layout: post
title: Reduce Chef infrastructure test times with test-kitchen and Docker
image: /images/posts/container-ships.jpg
tags:
- chef
- docker
- ops
---

Infrastructure automation tools like [Chef][CHE] and [Puppet][PUP] are becoming
more and more common as organizations look to code their infrastructure like
they code their applications.

At [Brigade][BRI], we use Chef to build our infrastructure. One of the challenges
we've faced is the tradeoff between thorough testing and the time it takes to
run those tests. With every Chef cookbook we write, we want to be able to
deploy changes with reasonable confidence that they will work in production. At
the same time, we don’t want to wait hours for automated scripts to spin up,
converge, and spin down nodes in order to verify that our code works.

One of the most powerful tools available in the Chef ecosystem is
[test-kitchen][TKI], a project that simplifies the process of converging your
cookbooks in an isolated test environment. Out of the box, it works by spinning
up a [VirtualBox][VBX] VM for each test suite, converging the VM with Chef, and
then running some tests to verify the VM is in a desired state.

# The Problem: Long Test Times

Where this solution has its limitations is that creating and destroying virtual
machines is time-consuming and resource-intensive. For our collection of over
20 test suites representing various core components of our infrastructure,
testing all of them took almost 2 hours. Converging and testing our base
cookbook (the cookbook included on all nodes in our infrastructure which
contains common global configuration) took almost 15 minutes alone, so even
running tests in parallel would still take significant time.

These long test times were not ideal because:

* We weren’t getting quick feedback on whether a commit broke our
  infrastructure in some way, which slowed down our development speed.
* Adding more test suites made our integration tests take even longer, so as
  our infrastructure grew the time it took to test grew at an unsustainable
  rate.

We clearly needed to explore other solutions to this problem.

# Identifying Bottlenecks

Digging into where we were spending significant amounts of time in our tests,
we realized that:

* Creating and destroying VMs took a minimum of 1–2 minutes for each test
  suite.

* There was noticeable CPU and disk I/O overhead running within a VM (5–10% in
  some cases).

* VMs limited our ability to parallelize, as each VM took a minimum amount of
  memory, even if it didn’t use all of it. This limited the utilization of our
  resources as a lot of memory would effectively go unused during the course of
  a test run---memory that could have been used to run other test suites.

While there certainly were ways we could have addressed some of these issues
while continuing to use VMs, it seemed like we needed to drastically rethink
how we were going about solving the problem, as the fundamental issue was the
overhead of running VMs.

# Solution: Containers as VMs

We realized that containers might be a viable alternative. Containers are quick
to provision, incur less overhead than a full VM, and make better use of
resources as they don’t reserve memory on startup, but share whatever system
memory is available. We decided to explore using [Docker][DKR] to provision
VM-like containers that we could run our tests within.

## kitchen-docker

Fortunately, there was already a [plugin][KDK] for test-kitchen that added
support for provisioning Docker containers. However, Docker containers are
usually very lightweight—they [typically aren't running SSH][SSH]. The
test-kitchen runtime however, relies on the existence of SSH in order to
connect to and run commands on a machine.

The kitchen-docker plugin handles this by generating a Dockerfile that runs an
SSH daemon within the container. This allows test-kitchen to access the
container via SSH and execute commands like running a Chef client and tests.
However, this only allows you to work with very simple Chef cookbooks—using
resource providers like [`service`][SVC] will cause Chef runs to fail since
there is no [init process][INT] running in the container.

## Thick Containers

In order to test our cookbooks that used the `service` resource provider, we
needed to come up with a way to get an init process running as the root process
in our containers. It seemed like we were trying to fit a square peg into a
round hole, and in many ways we were---Docker isn’t really intended to be
used for so-called "thick" containers, but it is perfectly capable of doing so.

Our infrastructure is built entirely on CentOS 7, whose init process is
[systemd][SYD]. Building on Jim Perrin's [Dockerfile][JPD] for a CentOS
container with systemd, we came up with the following Dockerfile:

{% highlight bash %}
# Defines Docker image suitable for testing cookbooks on CentOS 7.
#
# This handles a number of idiosyncrasies with systemd so it can be
# run as the root process of the container, making it behave like a
# normal VM but without the overhead.

FROM centos:centos7

# Systemd needs to be able to access cgroups
VOLUME /sys/fs/cgroup

# Setup container to run Systemd as root process, start an SSH
# daemon, and provision a user for test-kitchen to connect as.
RUN yum clean all && \
    yum -y swap — remove fakesystemd — install systemd systemd-libs && \

  # Remove unneeded unit files as this container isn't a proper machine
  (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ $i == systemd-tmpfiles-setup.service ] || rm -f $i; done) && \
  rm -f /lib/systemd/system/multi-user.target.wants/* && \
  rm -f /etc/systemd/system/*.wants/* && \
  rm -f /lib/systemd/system/local-fs.target.wants/* && \
  rm -f /lib/systemd/system/sockets.target.wants/*udev* && \
  rm -f /lib/systemd/system/sockets.target.wants/*initctl* && \
  rm -f /lib/systemd/system/basic.target.wants/* && \
  rm -f /lib/systemd/system/anaconda.target.wants/* && \

  # Setup kitchen user with passwordless sudo
  useradd -d /home/kitchen -m -s /bin/bash kitchen && \
  (echo kitchen:kitchen | chpasswd) && \
  mkdir -p /etc/sudoers.d && \
  echo 'kitchen ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers.d/kitchen && \

  # Setup SSH daemon so test-kitchen can access the container
  yum -y install openssh-server openssh-clients && \
  ssh-keygen -t dsa -f /etc/ssh/ssh_host_dsa_key -N '' && \
  ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key -N '' && \
  echo 'OPTIONS="-o UseDNS=no -o UsePAM=no -o PasswordAuthentication=yes"' >> /etc/sysconfig/sshd && \
  systemctl enable sshd.service

# Install basic system packages that we expect to exist by default.
# We do this in a separate RUN command since these packages are more
# likely to change over time, and we want to reuse previous layers as
# much as possible.
RUN yum -y install crontabs curl initscripts net-tools passwd sudo tar which && \
  (curl -L https://www.opscode.com/chef/install.sh | bash -s — -v 12.0.3)
{% endhighlight %}

Much of the above code is commented, but here are the following high-level
takeaways:

* We swap the fakesystemd dummy package included in the base CentOS 7 image
  with the actual systemd package, and remove some unnecessary unit files that
  only make sense for a "real" machine.
* We run SSH as a service under systemd.
* We finally install some basic utilities including Chef itself so that we
  don't need to install them each test run, saving us time.

The kitchen-docker configuration in our `.kitchen.yml` file is pretty
straightforward:

{% highlight yaml %}
driver:
  name: docker

platforms:
  - name: centos-7
    driver_config:
      dockerfile: test/Dockerfile
      privileged: true # Needed by systemd to access cgroups
      run_command: /usr/sbin/init # Start systemd as root process
      use_sudo: false
{% endhighlight %}

# Loose Ends

This got us 95% of the way towards our goal. However, two problems still remained:

* If our SSH daemon wasn't working for whatever reason, we could not log into the
machine via the kitchen login command.
* We noticed that destroying containers would timeout, resulting in a forced
shutdown which slowly [leaked loopback devices].

The first problem was solved by monkey-patching the kitchen-docker driver to
execute `docker exec -it %{container_id} bash` instead of SSH when using `kitchen
login`. Regardless of the state of the SSH daemon we could now always run a bash
shell within the container to poke around and debug.

{% highlight ruby %}
require 'kitchen/driver/docker'

module Kitchen
  module Driver
    class Docker < Kitchen::Driver::SSHBase
      def login_command(state)
        LoginCommand.new(%W[
          docker exec -it #{state[:container_id]} bash
        ])
      end
    end
  end
end
{% endhighlight %}

The second problem turned out to be a bit thornier. When Docker shuts down a
container, it sends the [SIGTERM][SGT] signal to the root process. Apparently,
systemd simply [re-execs][REX] itself when receiving SIGTERM, and there is no
other signal that causes it to shut down gracefully.

Thus we needed to monkey-patch the kitchen-docker driver to execute `shutdown
now` in the container to trigger a proper shutdown of systemd.

{% highlight ruby %}
require 'kitchen/driver/docker'

module Kitchen
  module Driver
    class Docker < Kitchen::Driver::SSHBase

      ...

      def rm_container(state)
        container_id = state[:container_id]
        docker_command("exec #{container_id} shutdown now")
        docker_command("wait #{container_id}") # Wait for shutdown
        docker_command("rm #{container_id}")
      end
    end
  end
end
{% endhighlight %}

This fixed our leaky loopback device woes.

In order to get these monkey-patches to work, we needed to include the
following line at the top of our `.kitchen.yml` file:


{% highlight yaml %}
# <% load "#{File.dirname(__FILE__)}/test/kitchen_docker.rb" %>
{% endhighlight %}

While it looks like it's in a comment, this actually gets executed since the
`.kitchen.yml` file is interpreted as ERB.

# The Results

The results of these efforts were substantial. Using the same test hardware as
before, our overall test time dropped from almost 2 hours to ~30 minutes, and
our base cookbook test suite dropped from 15 minutes to 3.

This was due to the containers being faster to create and destroy, as well as
having lower CPU overhead. We were also able to run more test suites in
parallel, since containers didn’t reserve memory all at once, but rather only
as needed.

## Gotchas

While this approach works for our infrastructure needs, this is not a silver bullet. Some important notes:

* The host machine running the container must also be CentOS 7 (or at least be
  running the same Linux kernel version).
* The containers need to be run with root privileges in order for the systemd
  within the container to access cgroups. We’re ok with this in our test
  infrastructure, but others might not like the idea of containers with root
  privileges on the host system.
* Perhaps obvious, but we’re not actually converging our cookbooks on a real
  machine. For our purposes this is fine, but there may be certain use cases
  for which you really need an isolated machine. In cases where you need true
  isolation, you can set the individual test suite to use the default
  test-kitchen Vagrant/VirtualBox integration.

[BRI]: https://www.brigade.com
[CHE]: https://www.chef.io/
[PUP]: https://puppetlabs.com/
[REL]: https://medium.com/brigade-engineering/smarter-relevant-chef-tests-ac998e957955
[TKI]: http://kitchen.ci/
[VBX]: https://www.virtualbox.org/
[DKR]: https://www.docker.com/
[KDK]: https://github.com/portertech/kitchen-docker
[SSH]: https://jpetazzo.github.io/2014/06/23/docker-ssh-considered-evil/
[SVC]: https://docs.chef.io/resource_service.html
[INT]: https://en.wikipedia.org/wiki/Init
[SYD]: http://www.freedesktop.org/wiki/Software/systemd/
[JPD]: https://jperrin.github.io/centos/2014/09/25/centos-docker-and-systemd/
[LLB]: https://github.com/jpetazzo/dind/issues/19
[SGT]: https://en.wikipedia.org/wiki/Unix_signal
[REX]: http://unix.stackexchange.com/a/7469
