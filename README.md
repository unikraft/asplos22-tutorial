# Unikraft ASPLOS'22 Tutorial Website

**To be held along with [International Conference on Architectural Support for
Programming Languages and Operating Systems
(ASPLOS'22)](https://asplos-conference.org)**.

With the advance of virtualization technology and the constant demand for
specialization, security and performance, unikernels are no longer a fringe
idea.  In this tutorial we present Unikraft, a unikernel SDK aiming for extreme
specialization.

Unikraft is an open source project with a growing and vibrant community
available on [GitHub](https://github.com/unikraft/) and
[Discord](https://bit.ly/UnikraftDiscord).

This tutorial is highly practical.  We will provide remote access to
pre-configured machines where attendees will build, configure, run and measure
Unikraft-based software components.  The tutorial is planned to take place for 6
hours (e.g. 10am to 5pm - 1 hour break).  Each section of this tutorial consists
of a short presentation / demo (10-15 minutes) followed by practical work to be
done by each attendee on their allocated remote machine.  Trainers from the
Unikraft community will provide instructions and support to attendees during the
tutorial.


## Running the website locally

Building and running the site locally requires a recent `extended` version of
[Hugo](https://gohugo.io). You can find out more about how to install Hugo for
your environment in our [Getting
started](https://www.docsy.dev/docs/getting-started/#prerequisites-and-installation)
guide.

Once you've made your working copy of the site repo, from the repo root folder, run:

```
hugo server
```

## Running a container locally

You can run this website inside a [Docker](https://docs.docker.com/) container,
the container runs with a volume bound to the `docsy-example` folder. This
approach doesn't require you to install any dependencies other than [Docker
Desktop](https://www.docker.com/products/docker-desktop) on Windows and Mac.

1. Build the docker image 

   ```bash
   TARGET=devenv make container
   ```

2. Run the built image

   ```bash
   make devenv
   ```

3. Once you are in the developer environment, you can call the hugo server
   command:

   ```
   hugo server --bind 0.0.0.0
   ```

4. Verify that the service is working. 

   Open your web browser and type `http://localhost:1313` in your navigation bar,
   This opens a local instance of the docsy-example homepage. You can now make
   changes to the docsy example and those changes will immediately show up in your
   browser after you save.
