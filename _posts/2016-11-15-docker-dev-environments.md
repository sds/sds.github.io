---
layout: post
title: "Dock: A tool to define, build, and run development environments with Docker"
tags:
- devops
- docker
---

At [Brigade][BRI], we’ve been using [Docker][DKR] both for development and
production with great success for the past two years. While a post about how we
use Docker in production is forthcoming, the tools we’ve built using Docker to
make our development lives easier and more effective is something particularly
worth sharing.

# Problems we were trying to solve

Not everyone has the same problems, and so what may work for our team may not
work for yours. The high-level problems were:

### Engineers weren’t able to quickly get a working development environment for our different services written in different languages requiring different runtime dependencies

We have a variety of projects written in Ruby, Scala, and Python, each with
slightly different dependencies (and potentially conflicting ones). Ideally,
installing all software dependencies and having a working instance of the
project would be as simple as running one command.

### Discrepancies between development and testing environments made reproducing failures difficult

We specifically had issues where a test would fail on our Jenkins build
machines but not locally, since most developers were on macOS whereas our build
machines were CentOS. Also, we wanted to enforce the particular versions of
software that developers were using (e.g. MySQL, Redis), which is difficult on
macOS, since most developers use [Homebrew][HMB] which always pulls the latest
versions by default (there are some workarounds such as [homebrew-versions][HBV],
but they are far from perfect).

Furthermore, due to the variety of projects we have, we needed to support
multiple “types” of build machines. We wanted to treat these machines like
cattle rather than pets and make them homogeneous to avoid dealing with
installing and managing conflicting dependencies of different projects.:w

### Testing major system changes in integration tests was time-consuming and potentially disruptive

Upgrading MySQL or Redis can be a daunting and stressful task. We have a
staging environment with production-like data, but in a company with teams
developing native apps against our staging environment, testing anything too
risky in staging is a recipe for blocking many engineers across the entire
company!

If we wanted to try out the upgrade in Jenkins builds first, upgrading backend
services on our Jenkins workers would involve modifying one of the workers
(making it a unique snowflake), running the tests on it and reverting the
machine back to the original state if the tests failed. This is a
time-consuming ordeal.

We wanted a solution that would ideally let us change a version in a
Dockerfile, have Jenkins test the commit, and get feedback on whether
anything broke.

# How Docker helped solve some of these problems

A common theme of all of these problems is the need to make something
reproducible, and to isolate environments from each other. These are both major
advantages offered by using Docker containers.

## Dock: Vagrant for Docker containers

Our solution was to create a tool called [dock][DCK]. It loads a `.dock`
configuration file from the root of a repository and uses it to create a
development environment with all the necessary dependencies available inside
the container, mounting the repository within it. If you’ve used
[Vagrant][VGT], this surely sounds familiar. We avoided Vagrant so we wouldn’t
require Ruby as a global dependency, keeping the tool as simple as possible and
easy to run in a wide variety of Docker images without adding to the image size
by including the Ruby runtime. Using containers is also significantly faster
than spinning up and down full virtual machines.

At a high level, Dock uses the `.dock` configuration file to create a [docker run
command][DRC] which it then executes, resulting in the developer being dropped into a
container with all the tools and services ready for development. If there is a
Dockerfile it needs to build to create the image, it builds that Dockerfile
first. Here’s an example of the resulting `docker run` command that is created
for one of our projects:

{% highlight bash %}
docker run \
  --name project-dock --hostname project-dock \
  --rm --interactive --tty --privileged \
  --workdir /workspace \
  --volume ~/src/project:/workspace:rw \
  --volume /usr/local/bin/dock:/usr/local/bin/dock \
  --volume project-dock_gems:/usr/local/bundle \
  --volume ~/src/project/script/dock-start-docker-daemon:/entrypoint.d/start-docker-daemon:ro \
  --volume project-dock_docker:/var/lib/docker:rw \
  --volume project_cache:/home/app/.cache \
  --volume project_local-storage:/home/app/.local \
  --volume ~/src/project/.my.cnf:/home/app/.my.cnf \
  --env INSIDE_DOCK=1 -p 3306:3306 -p 6379:6379 -p 9292:9292 \
  project:dock script/start-everything
{% endhighlight %}

Look how long this command is! It’s complex and difficult to grok at first
glance. The `.dock` configuration file responsible for this command is quite
long, but it’s well-documented and easier to maintain than a long list of flags
broken up over multiple lines. Here’s an excerpt of what the configuration
looks like with many lines removed for brevity (it’s just a Bash script that
Dock [sources][SRC]):

{% highlight bash %}
# File: /path/to/project/.dock

dockerfile Dockerfile # Use Dockerfile from the repository

# Store gems in a persistent volume
volume "$(container_name)_gems:/usr/local/bundle"

publish 3306:3306 # MySQL
publish 6379:6379 # Redis
publish 9292:9292 # Sinatra

# Command to execute when no explicit command passed to Dock
default_command script/start-everything
{% endhighlight %}

*Note: some of the commands (e.g. `container_name`) are helpers provided by Dock.*

Here’s the corresponding Dockerfile that’s referenced in the configuration
above:

{% highlight dockerfile %}
# File: /path/to/project/Dockerfile

FROM ruby:2.3.1

# Install Docker-related software
RUN curl -L https://get.docker.com/builds/Linux/x86_64/docker-1.12.3.tgz \
    | tar -xzf - -C /usr/local/bin --strip-components=1 \
    && curl -L https://github.com/docker/compose/releases/download/1.8.1/docker-compose-Linux-x86_64 --output /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/docker-compose

RUN yum install -y \
  # Allow us to compile mysql2 gem native extensions
  Percona-Server-client-56 Percona-Server-devel-56 \
  # Redis client
  redis \
  && yum clean all
{% endhighlight %}

Here’s an example of the output a user would see running Dock:

{% highlight dockerfile %}
$ cd src/project
$ dock
Building Dockerfile into image project:dock...
Sending build context to Docker daemon 3.584 kB
Step 1 : FROM ruby:2.3.1
2.3.1: Pulling from library/ruby

Digest: sha256:068ed8e5b7674ff899c1869b996498f6bef71ebcc6d30f56db510cc2505f2250
Status: Image is up to date for ruby:2.3.1
 ---> 813e5b0bd370
Step 2 : RUN curl -L https://get.docker.com/builds/Linux/x86_64/docker-1.12.3.tgz | tar -xzf - -C /usr/local/bin --strip-components=1     && curl -L https://github.com/docker/compose/releases/download/1.8.1/docker-compose-Linux-x86_64 --output /usr/local/bin/docker-compose && chmod +x /usr/local/bin/docker-compose
 ---> Using cache
 ---> cf874968fcdb
Step 3 : RUN yum install -y Percona-Server-client-56 Percona-Server-devel-56 redis && yum clean all
 ---> Using cache
 ---> 3cf899677bf6
Successfully built 3cf899677bf6
Dockerfile built into project:dock!
Starting container project-dock from image project:dock
...<script/start-everything command drops user into a shell here>...
{% endhighlight %}

The key takeaway here is that an engineer is able to get a working development
environment with no dependencies other than Docker and Bash by just running
dock in the project’s repository.

## Running scripts within a Dock environment

You might be saying “neat trick, but you’re still just executing `docker build`
and `docker run` commands,” and you’d be correct. However, another problem Dock
solves is making it easy to reproduce Jenkins builds exactly as they would be
on build machines, regardless of your host OS.

Dock accomplishes this by allowing you to add it to a script’s [shebang
line][SBG] so it is used as the interpreter. Here’s an example of a script that
runs RSpec tests:

{% highlight dockerfile %}
#!/usr/local/bin/dock bash

# Ensure we shut down MySQL cleanly before terminating
trap "docker-compose stop" EXIT INT QUIT TERM

docker-compose pull mysql
docker-compose up -d mysql

bundle install
bundle exec rake db:migrate
bundle exec rspec
{% endhighlight %}

Thanks to the shebang line, executing this script will result in it
automatically being run with Bash inside a Dock container (if we aren’t already
in one; if we are it just executes the script with Bash). This makes it simple
to check out a commit locally and run a script to quickly reproduce build
failures. A developer doesn’t even need to know that a script runs inside a
Dock container, as the shebang line automatically takes care of that for them.

You’ll also notice we’re using [docker-compose][DCP] to manage the MySQL
service that RSpec tests hit during the test run. This saves us having to
reinvent the wheel, as `docker-compose` does a fantastic job of allowing you to
define services and start/stop them as needed. This requires having a Docker
daemon running inside the container (a.k.a. “Docker-in-Docker”), which comes
with a [number of gotchas][GTC], but we’ve used it with great success.

## Running services inside the Dock container

Running a Docker daemon inside the container gives us a lot of power, since we
can now stand up services that are isolated to that container. This allows you
to run multiple projects simultaneously even if they run services that listen
on the same port (e.g. MySQL or Redis). You could configure these to use
different ports for each of your projects, but then you need to keep track of
all of these ports. Using well-known ports is easier since it requires less
configuration, and eases the cognitive load on engineers.

<figure>
  <img src="/images/posts/dock-architecture.png"
       title="Docker daemon running services in a Dock container via docker-compose"
       width="80%">
</figure>

Inside our Dock containers, we use `docker-compose` to spin up the various
services needed by a project. Here’s a `docker-compose.yml` for a project that
depends on Redis:

{% highlight yaml %}
version: '2'

services:
  redis:
    image: 'redis:2.8'
    network_mode: host
    volumes:
      - 'redis:/data'

# Define names of volumes that you want to be preserved between
# container restarts.
# These are referenced above in the `volumes` option of a service.
volumes:
  redis:
{% endhighlight %}

If you run `docker-compose up -d` in the Dock container, Redis will be started
in a container in the background. Notice that we specify
`network_mode: host` — since we’re running the Redis container with the Docker
daemon running inside the Dock container, the network namespace is already
isolated. This removes the need to specify the `port` option for services since
we don’t need to proxy from outside the container to inside the container.

We can automate this so that when you run dock inside the project we
automatically start up all the needed background services. Recall the last line
of our `.dock` configuration file:

{% highlight bash %}
# Command to execute when no explicit command passed to Dock
default_command script/start-everything
{% endhighlight %}

We can have `script/start-everything` run `docker-compose up -d` for us.
A simplified example:

{% highlight bash %}
# File: /path/to/project/script/start-everything

#!/usr/local/bin/dock bash

# Ensure we shut down Redis cleanly before terminating
trap "docker-compose stop" EXIT INT QUIT TERM

docker-compose pull redis
docker-compose up -d redis

# Drop into a shell
bash -il
{% endhighlight %}

The above script allows us to add custom logic for starting up our development
environment, using nothing but Bash and the Docker tool set.

This has proven incredibly useful at Brigade. It means engineers can get up and
running on their laptop by installing Docker and running a single command,
`dock`, without any additional setup.

# Problems we haven’t solved yet

We won’t lie to you and say everything is now sunshine and rainbows. While
Docker helped solve many of the problems above, we still face challenges.
However, we think the net result is a win for our team’s productivity and
ability to iterate quickly, as well as lowering the barrier for engineers to
jump between different projects.

Examples of issues we still have:

### Problem 1: Different Docker images between development and production

Since development environments often include a number of additional tools used
in debugging and regular development which you wouldn’t include in production,
we still maintain different images for deploying our applications to
production. This hasn’t been a huge issue but it would be nice to know that the
software we test locally uses the exact same image as what we deploy to
production, as this ensures fewer possible ways for our code to work in
development but not production. At the end of the day, it feels somewhat
impractical given the importance of developer time, and so having a
development/testing-specific environment that optimizes for developer time is a
pragmatic tradeoff here.

### Problem 2: Starting up dependent services in different repos

When developing microservices, you’ll inevitably have one service make a call
to another. While we usually test these calls against our staging environment,
sometimes one needs to make changes to both services, which makes this
difficult to test unless you can run them both locally. Since we run the
service inside the Dock container, this makes it difficult to link with other
services running inside a different Dock container, since they are both
isolated from the host network.

We don’t yet have a good solution for this outside of deploying the change to
staging and testing it out, risking a break in staging in the process. We
haven’t bothered solving this problem because needing to run both services
locally is rare for us.

# Let us know what you think

How are you using Docker to optimize your development and testing processes? If
you’re interested in giving Dock a try or suggesting improvements, feel free to
[check out the project][DCK]!

[BRI]: https://www.brigade.com
[DKR]: https://www.docker.com
[HMB]: http://brew.sh
[HBV]: https://github.com/Homebrew/homebrew-versions
[DCK]: https://github.com/brigade/dock
[VGT]: https://www.vagrantup.com
[DRC]: https://docs.docker.com/engine/reference/run/
[SRC]: http://www.tldp.org/HOWTO/Bash-Prompt-HOWTO/x237.html
[SBG]: https://en.wikipedia.org/wiki/Shebang_%28Unix%29
[DCP]: https://docs.docker.com/compose/
[GTC]: https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/
