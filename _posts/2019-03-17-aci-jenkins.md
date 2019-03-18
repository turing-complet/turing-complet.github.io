---
layout: post
title:  "Running jenkins on azure container instances"
date:   2019-03-17 00:26:54 -0700
categories: jekyll update
---
I wanted to document how I spent too much time on a build system instead of writing code, so this is that happening, right now.

Discliamer - this is far from being a complete solution to CI/CD. It's a way for me to have a bit more control over my tools without going overboard.

The motivation for this is the following:
* I want to trigger builds when I push code to certain repositories
* I don't always have builds to publish/deploy/etc, so when I do, I want to spin up the build server quickly
* Artifacts - configuration, docker images, etc, should stick around afterwards
* I wanted something that could potentially be used for real projects if needed
* Should be container oriented to minimize environmental concerns

With that out of the way, here's the real stuff.

The base for this will be the jenkins blue ocean docker image, seen [here](https://hub.docker.com/r/jenkinsci/blueocean/).
This makes it easy to define a build in the UI, then save it to the source repo in a Jenkinsfile which expresses the build as code. 

Next we will need to run this. I used an azure container instance (ACI) because it's the easiest way I could find to run a single container (and attach a public ip/dns to it, though this isn't strictly necessary since jenkins polls github).

I wanted to use terraform to manage this instance, but since ACI is still new, it's not supported yet, so I just used the azure cli commands in a powershell script. I will go through how the script works though it's mostly self explanatory, then show how jenkins is configured. I couldn't think of a clever way to automate that configuration but it's a one time thing that will persist in a storage account, so it's probably fine.

Okay, so back to creating a container instance. First we need a resource group and some environment variables:
```
az login
az group create --name jenkins --location westus2
$random = [System.Guid]::NewGuid().ToString("N").Substring(5, 10)
$env:ACI_PERS_RESOURCE_GROUP = "jenkins"
$env:ACI_PERS_STORAGE_ACCOUNT_NAME = "jenkins" + $random
$env:ACI_PERS_LOCATION = "westus2"
$env:ACI_PERS_SHARE_NAME = "acishare"
```
This allows us to create the storage account and file share:

```
# Create the storage account
az storage account create `
    --resource-group $env:ACI_PERS_RESOURCE_GROUP `
    --name $env:ACI_PERS_STORAGE_ACCOUNT_NAME `
    --location $env:ACI_PERS_LOCATION `
    --sku Standard_LRS

# Create the file share
az storage share create --name $env:ACI_PERS_SHARE_NAME --account-name $env:ACI_PERS_STORAGE_ACCOUNT_NAME
```

Now to create the container, we will need the key to access the storage resource, which we can get with this query that I copied from azure docs.

```
$env:STORAGE_KEY=(az storage account keys list `
    --resource-group $env:ACI_PERS_RESOURCE_GROUP `
    --account-name $env:ACI_PERS_STORAGE_ACCOUNT_NAME `
    --query "[0].value" `
    --output tsv)
```

Nice. Now pick a dns name (I'll use aci-build-demo) and we're ready to go:

```
az container create `
    --resource-group $env:ACI_PERS_RESOURCE_GROUP `
    --name jenkins `
    --image jenkinsci/blueocean `
    --dns-name-label aci-build-demo `
    --ports 8080 `
    --azure-file-volume-account-name $env:ACI_PERS_STORAGE_ACCOUNT_NAME `
    --azure-file-volume-account-key $env:STORAGE_KEY `
    --azure-file-volume-share-name $env:ACI_PERS_SHARE_NAME `
    --azure-file-volume-mount-path /var/jenkins_home
```
You can get the public address in the portal or just use

```
az container show --resource-group $env:ACI_PERS_RESOURCE_GROUP --name jenkins --query ipAddress.fqdn
```

In my case, it is at http://aci-build-demo.westus2.azurecontainer.io:8080/. This can all be wrapped in a script, so it's just one command to spin up a build server. Wow. Such technology.

This thing isn't useful though unless we configure it to do things. So let's do that. First step is create a github token and give it to jenkins so it can access your code. This can be done in https://github.com/settings/tokens under "personal access tokens."


![Github token]({{ site.url }}/assets/github-token.png)

Now login to jenkins using the password which will be in the docker logs. If you can't find it there, it's stored in `/var/jenkins_home/secrets/initialAdminPassword` in the container. 

Open blue ocean and create a new pipeline, pasting in the github token. You should see your repositories:

![Repos]({{ site.url }}/assets/github-connect.png)

We have connection! Now we're cooking with \<your favorite cooking fuel\>. 

Before we proceed though, I'm going to gloss over a major part of this - I use jenkins to kick off docker builds, which requires docker engine. In principle we could have azure start "sibling" builds (rather than use nested docker, which is [not ideal](https://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/)), but since ACI is a managed service, accessing the docker socket is not allowed. Therefore, I used a jenkins [plugin](https://docs.microsoft.com/en-us/azure/jenkins/jenkins-azure-vm-agents) to automatically provision VM build agents with docker engine on them to run the builds. This requires a one time configuration through the jenkins ui. Perhaps there is a way to make it a template (parametrized credentials, obviously) but I haven't gotten there yet. Once that's squared away, we can get to running builds.

Note: we need to set "docker" as a label on the VM agents, so jenkins will use them.

Going back to the blue ocean ui, we can define a new build by using the pipeline editor (see jenkins [docs](https://jenkins.io/doc/tutorials/create-a-pipeline-in-blue-ocean/)) which apparently contain much of what I've just written about. Anyway, my pipeline now looks like this:

![Pipeline]({{ site.url }}/assets/pipeline.png)

The same pipeline can be found in a Jenkinsfile [here](https://github.com/turing-complet/dockerfiles/blob/master/Jenkinsfile). Now merges to master will push docker images to my azure image registry, which is so easy to setup it's not worth writing about here.

That's pretty much it for now. To conclude, we can spin up a jenkins instance with one command, configure it once to connect to github, auto provision vm agents, store that configuration in a persistent share, and delete the containers when not needed to stop incurring charges. Hopefully this will become more automated so anyone could spin up an entire build system in seconds, but that depends on my attention span...