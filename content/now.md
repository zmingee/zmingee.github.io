---
title: What I'm Doing Now
date: 2021-12-27
type: post
---

[This is a now page](https://nownownow.com/about)

# Development

## Learning Go

I've been expanding my basic knowledge of Go. Mainly working with CLIs (Cobra),
configuration (Viper), directory structure, building libraries and modules,
using open APIs to learn `net/http`. I'm fluent in Python, though, so it will
take some work to get comfortable with Go.

# Systems

## Learning Kubernetes, K3s, k3OS, Helm, etc

I've been diving into the Kubernetes world for the last couple of months.
I started with a few Raspberry Pis running Alpine Linux and K3s. I've since
moved onto k3OS, which is much friendlier to use and configure. Since I've
learned how to write manifests and deploy applications that way, I'm now
learning Helm.

I've been diving into the Kubernetes world for the last few months, but it's
accelerated the last month or two. It started with a few Raspberry Pi's, and
then grew from there. I didn't want to try and learn all of Kubernetes at one
time, so I got started with K3s. 

# Workstations

A recent SSD failure forced me to better manage my backups, home directories,
configs, and dotfiles between my workstations. I'm using:

- [BorgBackup](https://www.borgbackup.org/)
- [Unison](https://www.cis.upenn.edu/~bcpierce/unison/)
- [Stow](https://www.gnu.org/software/stow/)

I've already been using these tools, but now it's automated and done
"properly". I also wrote some glue code to get all of them automated and
audited. I may look into using an Ansible playbook to manage more of the system
config.
