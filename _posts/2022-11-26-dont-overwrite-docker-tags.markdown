---
layout: post
title:  "Don't overwrite docker tags"
date:   2022-11-26 14:51:22 +0100
categories: docker devops
---

# The ambiguity of docker tags
You might have already heard that using the [:latest tag in docker is bad practise](https://vsupalov.com/docker-latest-tag/). This is true for several reasons. For starters, a new image might be pushed without adding the latest tag. Meaning that if you pull image:latest, it might not actually be the latest image.

On top of that, if you're running a new container, you might not really be using the latest image, even if you're using the latest tag. Not all tools will automatically check if a new version is available. If you already have an image tagged as `latest` locally, docker will not automatically pull a new image.

I'd like to extend this reasoning for other tags. Let's say you already pulled `image:0.0.1`, but someone else pushed a new image using the same tag. Next time you execute `docker run image:0.0.1`, it will not automatically pull the updated image. You'll still be using the 'old' version 0.0.1. 

This problem can be avoided by using `docker run --pull=always`, but what about containers that are already running? If you're not careful, you will have two different versions of version 0.0.1 running. If that sentence sounds confusing, that's because it is. 

Don't overwrite docker tags.


# Automated madness

I wouldn't be writing this if I hadn't recently run into this problem. Our container registry allowed to overwrite docker images, and our continuous integration pipelines (CI/CD) did not automatically bump versions. This was a project with very little changes once it was set up, and any change usually was accompanied with a redeployment of most of our infrastructure. 

This was fine for the first couple of months, but then Dependabot was introduced on here, as well as on some other projects. It started occasionally opening pull requests, which in turn automatically triggered a run on our CI/CD. 

I already talked about bad practise: allowing overwriting docker images, but that in itself would not have caused problems. There was a bug in our CI/CD, that also published docker images (under the same tag) when running checks on Pull Requests (PR).  Since there were so few changes, and all changes were usually immediately accepted, this remained unnoticed for a long time. 

Dependabot automatically updated some dependencies, this triggered an automated build of the project, which was automatically published on an existing tag. 

The final piece of the puzzle was Kubernetes, where our application was running. Kubernetes will not automatically redeploy a new image when it's available, but it will always pull the image when starting a new container. Several days after the PR from Dependabot was opened, Kubernetes had to restart a failed pod, and pulled the new image. 

At this point we had two different instances of the same application running side by side, but looking at the pod definition, they were identical. This whole process did not involve any human interaction. 

# Mitigations 

While this could have been avoided by not publishing builds when running CI/CD on pull requests, the real issue was pushing images with tags that already existed.

Unfortunately Azure Container Registry (ACR), where our images are hosted, does not allow automatically locking image tags once in use, but [you can lock existing tags individually](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-image-lock#lock-an-image-by-tag):

```
az acr repository update \
    --name myregistry --repository myrepo \
    --write-enabled false
```

It can be a good idea to run this as part of your CI/CD each time an image is published. 
With some registries, such as [JFrog Artifactory](https://jfrog.com/artifactory/) you can [prevent overwriting existing deployments](https://www.jfrog.com/confluence/display/RTF6X/Managing+Permissions).

