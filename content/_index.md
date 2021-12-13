---
title: 'Overview'
date: 2018-11-28T15:14:39+10:00
weight: 1
---

## Unikraft ASPLOS'22 Tutorial

**To be held along with [International Conference on Architectural Support for
Programming Languages and Operating Systems
(ASPLOS'22)](https://asplos-conference.org)**.

With the advance of virtualization technology and the constant demand for
specialization, security and performance, unikernels are no longer a fringe
idea.  In this tutorial we present Unikraft, a unikernel SDK aiming for extreme
specialization.

Unikraft consists of core software components (i.e. internal libraries) and
outside optional components (i.e. external libraries), most of which are
existing software components (standard C libraries, runtime libraries,
frameworks, applications) integrated with the core parts.  By allowing extreme
configurability and using common APIs backed by multiple implementations,
Unikraft makes it easy to tailor custom software solutions for your use case: be
it performance, reduced resource utilization, security and safety or
functionality, Unikraft makes it happen.

Unikraft is an open source project with a growing and vibrant community
available on [GitHub](https://github.com/unikraft/) and
[Discord](https://bit.ly/UnikraftDiscord).  The community consists of highly
skilled industry contributors, academics and students, motivated in creating
next-generation software products, pursuing novel research objectives and
developing open and challenging technology in operating systems.  Recent works
realized on the basis of Unikraft include CubicleOS (ASPLOS'21) and FlexOS (to
be separately presented at ASPLOS'22).

In this tutorial, we aim to get attendees accustomed to Unikraft and unikernel
technology.  We present Unikraft internals, highlight research prospects and
showcase performance and specialization use cases. We start with an introduction
in unikernels and Unikraft, details the build and configuration process of
Unikraft, the porting / integration of libraries and applications (both in
source code and in binary form) and performance-oriented scenarios.

The tutorial will be highly practical.  We will provide remote access to
pre-configured machines where attendees will build, configure, run and measure
Unikraft-based software components.  Attendees are required to have a laptop and
an SSH client to access the remote system. Attendees are expected to have good
Linux/CLI/shell scripting skills, experience with C programming and build
automation and general knowledge of virtualization technology.

Each section in the tutorial will consist of a short presentation / demo (10-15
minutes) followed by practical work to be done by each attendee on their
allocated remote machine.