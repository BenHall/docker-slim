# docker-slim: Lean and Mean Docker containers

[Docker Global Hack Day \#dockerhackday](https://www.docker.com/community/hackathon) project (status: ACTIVE!)

Just because the hack day is over doesn't mean the project is done :-) The project needs your help even if you don't know Docker or Go!

IRC (freenode): \#dockerslim

## NEW

Auto-generated seccomp profiles for Docker 1.10. [Latest binaries](https://github.com/cloudimmunity/docker-slim/releases/download/1.11/dist_mac.zip)

## DEMO VIDEO

[![DockerSlim demo](http://img.youtube.com/vi/uKdHnfEbc-E/0.jpg)](https://www.youtube.com/watch?v=uKdHnfEbc-E)

[Demo video on YouTube](https://youtu.be/uKdHnfEbc-E)

## DESCRIPTION


Creating small containers requires a lot of voodoo magic and it can be pretty painful. You shouldn't have to throw away your tools and your workflow to have skinny containers. Using Docker should be easy. 

`docker-slim` is a magic diet pill for your containers :) It will use static and dynamic analysis to create a skinny container for your app.

## CURRENT STATE

It WORKS with the sample node.js, python, and ruby images (built from `sample_apps`). More testing needs to be done to see how it works with other images.

Sample images (built with the standard Ubuntu 14.04 base image):

* nodejs app container: 431.7 MB => 14.22 MB
* python app container: 433.1 MB => 15.97 MB
* ruby app container:   406.2 MB => 13.66 MB
* java app container:   743.6 MB => 100.3 MB (yes, it's a bit bigger than others :-))

You can also run `docker-slim` in the `info` mode and it'll generate useful image information including a "reverse engineered" Dockerfile.

DockerSlim now also generates AppArmor and Seccomp profiles for your container.

Works with Docker 1.8, 1.9 and 1.10.

Dependencies:

To run `docker-slim` you need to export docker environment variables. If you use `docker-machine` you get it when you run `eval "$(docker-machine env default)"`.

## FAQ

### Is it safe for production use?

Yes! Either way, you should test your Docker images.

### How can I contribute if I don't know Go?

You don't need to read the language spec and lots of books :-) Go through the [Tour of Go](https://tour.golang.org/welcome/1) and optionally read [50 Shades of Go](http://devs.cloudimmunity.com/gotchas-and-common-mistakes-in-go-golang/) and you'll be ready to contribute!

### What's the best application for DockerSlim?

DockerSlim will work for any dockerized application; however, DockerSlim automates app interactions for applications with an HTTP API. You can use DockerSlim even if your app doesn't have an HTTP API. You'll need to interact with your application manually to make sure DockerSlim can observe your application behavior.

## USAGE

`./docker-slim [info|build] [--http-probe|--remove-file-artifacts] <IMAGE_ID_OR_NAME>`

Example: `./docker-slim build --http-probe my/sample-node-app`

To generate a Dockerfile for your "fat" image without creating a new "slim" image use the `info` command.

Example: `./docker-slim info 6f74095b68c9`

## DEMO STEPS

The demo run on Mac OS X, but you can build a linux version. Note that these steps are different from the steps in the demo video.

0. Get the docker-slim [binaries](https://github.com/cloudimmunity/docker-slim/releases/download/1.11/dist_mac.zip). Unzip them and optionally add their directory to your PATH environment variable if you want to use the app from other locations.

	The extracted directory contains two binaries:

	* `docker-slim` <- the main application
	* `docker-slim-sensor` <- the sensor application used to collect information from running containers

1. Clone this repo to use the sample apps. You can skip this step if you have your own app.

	`git clone https://github.com/cloudimmunity/docker-slim.git`
	
2. Create a Docker image for the sample node.js app in `sample/apps/node`. You can skip this step if you have your own app.
	
	`cd docker-slim/sample/apps/node`
	
	`eval "$(docker-machine env default)"` <- optional (depends on how Docker is installed on your machine); if the Docker host is not running you'll need to start it first: `docker-machine start default`
	
	`docker build -t my/sample-node-app .`
	 
3. Run `docker-slim`:
	
	`./docker-slim build --http-probe my/sample-node-app` <- run it from the location where you extraced the docker-slim binaries
	
	DockerSlim creates a special container based on the target image you provided. It also creates a resource directory where it stores the information it discovers about your image: `<docker-slim directory>/.images/<TARGET_IMAGE_ID>`.

4. Use curl (or other tools) to call the sample app (optional)

	`curl http://<YOUR_DOCKER_HOST_IP>:<PORT>`
	
	This is an optional step to make sure the target app container is doing something. Depending on the application it's an optional step. For some applications it's required if it loads new application resources dynamically based on the requests it's processing.
		
	You can get the port number either from the `docker ps` or `docker port <CONTAINER_ID>` commands. The current version of DockerSlim doesn't allow you to map exposed network ports (it works like `docker run … -P`).

	If you set the `http-probe` flag then `docker-slim` will try to call your application using HTTP/HTTPS: `./docker-slim build --http-probe my/sample-node-app`

5. Press any key and wait until `docker-slim` says it's done

6. Once DockerSlim is done check that the new minified image is there

	`docker images`
	
	You should see `my/sample-node-app.slim` in the list of images. Right now all generated images have `.slim` at the end of its name.

7. Use the minified image

	`docker run --name="slim_node_app" -p 8000:8000 my/sample-node-app.slim`

Notes:

You can explore the artifacts DockerSlim generates when it's creating a slim image. You'll find those in `<docker-slim directory>/.images/<TARGET_IMAGE_ID>/artifacts`. One of the artifacts is a "reverse engineered" Dockerfile for the original image. It'll be called `Dockerfile.fat`. 

If you'd like to see the artifacts without running `docker-slim` you can take a look at the `sample/artifacts` directory in this repo. It doesn't include any image files, but you'll find:

*	a reverse engineered Dockerfile (`Dockerfile.fat`)
*	a container report file (`creport.json`)
*	a sample AppArmor profile (which will be named based on your original image name)
*   and a sample Seccomp profile

If you don't want to create a minified image and only want to "reverse engineer" the Dockerfile you can use the `info` command.

## USING AUTO-GENERATED SECCOMP PROFILES

You can use the generated profile with your original image or with the minified image DockerSlim created:

`docker run --security-opt seccomp:path_to/my-sample-node-app-seccomp.json -p 8000:8000 my/sample-node-app.slim`

## BUILD PROCESS

Go 1.5.1 or higher is required. Earlier versions of Go have a Docker/ptrace related bug (Go kills processes if your app is PID 1). When the 'monitor' is separate from the 'launcher' process it will be possible to user older Go versions again.

Before you build the tool you need to install GOX and Godep (optional; you'll need it only if you have problems pulling the dependencies with vanilla `go get`)

* Godep - dependency manager ( https://github.com/tools/godep )

1: `go get github.com/tools/godep`

* GOX - to build Linux binaries on a Mac ( https://github.com/mitchellh/gox ):

1: `go get github.com/mitchellh/gox`

2: `gox -build-toolchain -os="linux" -os="darwin"` (note:  might have to run it with `sudo`)

Note:

Step 2 is not necessary with Go 1.5.

#### Local Build Steps

Once you install the dependencies (GOX - required; Godep - optional) run these scripts:

1. Pull the dependencies: `./scripts/src.deps.get.sh`
2. Build it: `./scripts/src.build.sh`

You can use the clickable `.command` scripts on Mac OS X (located in the `scripts` directory):

1. `mac.src.deps.get.command`
2. `mac.src.build.command`

#### Traditional Go Way to Build

If you don't want to use the helper scripts you can build `docker-slim` using regular go commands:

1. `cd $GOPATH`
2. `mkdir -p src/github.com/cloudimmunity`
3. `cd $GOPATH/src/github.com/cloudimmunity`
4. `git clone https://github.com/cloudimmunity/docker-slim.git` <- if you decide to use `go get` to pull the `docker-slim` repo make sure to use the `-d` flag, so Go doesn't try to build it
5. `cd docker-slim`
6. `go get -d -v ./...`
7. `go build -v ./apps/docker-slim` <- builds the main app in the repo's root directory
8. `env GOOS=linux GOARCH=amd64 go build -v ./apps/docker-slim-sensor` <- builds the sensor app (must be built as a linux executable)

#### Builder Image Steps

You can also build `docker-slim` using a "builder" Docker image. The helper scripts are located in the `scripts` directory.

1. Create the "builder" image: `./docker-slim-builder.build.sh` (or click on `docker-slim-builder.build.command` if you are using Mac OS X)
2. Build the tool: `docker-slim-builder.run.sh` (or click on `docker-slim-builder.run.command` if you are using Mac OS X)

## DESIGN

### CORE CONCEPTS

1. Inspect container metadata (static analysis)
2. Inspect container data (static analysis)
3. Inspect running application (dynamic analysis)
4. Build an application artifact graph
5. Use the collected application data to build small images
6. Use the collected application data to auto-generate various security framework configurations.

### DYNAMIC ANALYSIS OPTIONS

1. Instrument the container image (and replace the entrypoint/cmd) to collect application activity data
2. Use kernel-level tools that provide visibility into running containers (without instrumenting the containers)
3. Disable relevant namespaces in the target container to gain container visibility (can be done with runC)

### SECURITY

The goal is to auto-generate Seccomp, AppArmor, (and potentially SELinux) profiles based on the collected information.

* AppArmor profiles
* Seccomp profiles

### CHALLENGES

Some of the advanced analysis options require a number of Linux kernel features that are not always included. The kernel you get with Docker Machine / Boot2docker is a great example of that.


## DEVELOPMENT PROGRESS

### PHASE 1 (DONE)

Goal: build basic infrastructure

Create the "slim" app that:

*  collects basic container image metadata [DONE]
*  "reverse engineers" the Dockerfile used to create the target image [DONE]
*  creates a container replacing/hooking the original entrypoint/cmd [DONE]
*  creates a new "slim" image from the collected information and artifacts [DONE]

Create the "slim" launcher that:

* starts the original application (based on the original entrypoint/cmd data) [DONE]
* monitors process activity (saving events in a log file) [DONE] (note: doesn't work with all kernels)
* monitors file activity (saving events in a log file) [DONE]

### PHASE 2 (DONE)

* Fix new image permission errors [DONE]
* Use env data from the original image [DONE]

### MILESTONE 1 - MINIFIED TEST DOCKER IMAGE (DONE)

The minified `sample_app` docker image now works! We turned a 430MB node.js app container into a 40MB image.

### PHASE 3 (ACTIVE)

* Do a better job with links [DONE] The test image is now even smaller (was: 40MB, now: 14.22MB)
* Make sure it works with other images [WIP, now: node,python,ruby,java].
* Refactor the time-based container monitoring phase [DONE].
* Automated interaction with the target container (requires app code analysis) [WIP;DONE - simple version].
* Auto-generate AppArmor profiles [WIP].
* Auto-generate Seccomp filters [USABLE, Docker 1.10+ is required].
* Split "monitor" from "launcher" (as it's supposed to work :-))
* Add scripting language dependency discovery to the "scanner" app.
* Support additional command line parameters to specify CMD, VOLUME, ENV info.
* Build/use a custom Boot2docker kernel with every required feature turned on.
* Explore additional dependency discovery methods.
* "Live" image create mode - to create new images from containers where users install their applications interactively.


## NOTES

1. The code is still pretty ugly at this point in time :)
2. Each app directory contains a dummy `.git` directory because `godep` fails to work without it.






