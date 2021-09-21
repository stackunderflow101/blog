---
title: Containers Deep Dive
author: Stack Underflow
date: '2021-09-21'
slug: containers-deep-dive
categories:
  - Tech
tags:
  - containers
description: A collection of resources on containers
---

## The underlying technology about containers

The series [Namespaces in operation](https://lwn.net/Articles/531114/) on LWN covers the underlying technology about containers: Linux namespaces. Some code examples are outdated because of new Linux kernel releases and the author provides updates in the comments. If the programs in the article do not work, do refer to the comments to see what should be modified.

The author also has a two-videos series about namespaces:

- [Containers unplugged: Linux namespaces](https://youtu.be/0kJPa-1FuoI)
- [Containers unplugged: understanding user namespaces](https://youtu.be/73nB9-HYbAI)

## The containers ecosystem

The article series [Dockerless](https://mkdev.me/en/posts/dockerless-part-1-which-tools-to-replace-docker-with-and-why) explains tools other than Docker to work with containers. The course advertised in the articles [DOCKERLESS: Re-explore containers from open standards perspective](https://courses.mkdev.me/p/dockerless-re-explore-containers-from-open-standards-perspective) (note: 💲) explains the topics in more detail, including:

- Low level container standard (OCI) and tools (like `runc`)
- Tools other than Docker to work with containers (like `buildah` and `podman`)

Although already mentioned in the Dockerless series, I want to highlight here that the articles from [Daniel J Walsh](https://opensource.com/users/rhatdan) are great. For example, the series about rootless containers explains how Podman runs containers rootlessly using user namespaces (In contrast, Docker by default does not use user namespaces and requires root privilege):

- [Podman and user namespaces: A marriage made in heaven](https://opensource.com/article/18/12/podman-and-user-namespaces)
- [How does rootless Podman work?](https://opensource.com/article/19/2/how-does-rootless-podman-work)
- [How rootless Buildah works: Building containers in unprivileged environments](https://opensource.com/article/19/3/tips-tricks-rootless-buildah)
- [The shortcomings of rootless containers](https://opensource.com/article/19/5/shortcomings-rootless-containers)

