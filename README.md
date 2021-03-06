# Deploying Meteor Apps on Cloud Foundry

This repository demonstrates different ways to deploy Meteor applications on Cloud Foundry. 
There are at least three different ways to deploy Meteor JS applications on Cloud Foundry:

  - using a specialized meteor-js buildpack
  - using the general node-js buildpack
  - using your own docker container image

While a specialized meteor buildpack like [meteor-buildpack-horse](https://github.com/AdmitHub/meteor-buildpack-horse) or [cf-meteor-buildpack](https://github.com/cloudfoundry-community/cf-meteor-buildpack) is a useful options, recent versions of meteor support simplified deployment on servers that provide just a node.js runtime environment. For this demonstration repository, we're focusing on this approach because the node.js buildpack is more widely used and better maintained. 

The application we're going to deploy is located in the `./try-meteor` folder of this repository. We also assume you have provisioned a Mongo DB service on your space. At [meshcloud](https://panel.meshcloud.io/)'s Cloud Foundry, you can create a dedicated service instance suitable for use with meteor like this:

```bash
cf create-service MongoDB M meteor-mongo
```

## Using the node.js Buildpack

### Build a meteor app bundle
On your local machine with the meteor cli installed, build a distributable package into the `deploy-buildpack` folder:

```bash
cd try-meteor && meteor build ../deploy-buildpack/. --server-only --architecture os.linux.x86_64
```

Building generates a `try-meteor.tar.gz` file with the meteor application bundled as a plain node.js application, with some helfpful instructions in its `README.md` file: 
```
This is a Meteor application bundle. It has only one external dependency:
Node.js v8.9.3. To run the application:

  $ (cd programs/server && npm install)
  $ export MONGO_URL='mongodb://user:password@host:port/databasename'
  $ export ROOT_URL='http://example.com'
  $ export MAIL_URL='smtp://user:password@mailhost:port/'
  $ node main.js

Use the PORT environment variable to set the port where the
application will listen. The default is 80, but that will require
root on most systems.
```

### node.js buildpack wrapper
To deploy this tar.gz file on Cloud Foundry with the node.js buildpack we need to: 

  1. upload and unpack the tar.gz bundle
  2. run `npm install` on the extracted bundle
  3. set the correct environment variables using a launcher js script

We can easily achieve that through a custom `package.json` that uses npm's `postinstall` and `start` script to execute these actions. You can find the `package.json` and all required files for the deployment in the `./deploy-buildpack` folder.

> Note: at the time of writing the bundles generated by meteor 1.6.0.1  machine lack the `meteor-deque` dependency so we just explicitly add that by hand.

```json
{
  "name": "try-meteor",
  "private": true,
  "scripts": {
    "start": "node launcher.js",
    "postinstall": "tar -xf try-meteor.tar.gz && (cd bundle/programs/server && npm install)"
  },
  "engines" : {
    "node" : "8.9.3"
  },
  "dependencies": {
    "meteor-deque": "~2.1.0",
    "cfenv": "1.0.4"
  }
}
```

Have a look at the `launcher.js` file if you want to change service names etc.
The final bit that we need is a Cloud Foundry Manifest file to describe our application: 

```yml
---
applications:
- name: try-meteor-app
  memory: 512M
  instances: 1
  buildpack: https://github.com/cloudfoundry/nodejs-buildpack
  services:
    - meteor-mongo
```

We're all set now, a simple `cf push` and your app should be up and running on the cloud. 

## Using a Docker Container

The next option is to use a docker-based deployment of the application. This requires that we build our own docker image of the application and publish it to a docker registry.

You can find the code for the docker-based deployment of our sample application in the `./deploy-docker` folder. The docker image used in this example is based on the [node:8-alpine](https://hub.docker.com/_/node/) base image.  However, before we can build our container we need to build our meteor application and extract it:

```bash
cd try-meteor && meteor build ../deploy-docker/. --server-only --architecture os.linux.x86_64
cd ../deploy-docker && tar -xf try-meteor.tar.gz  && rm try-meteor.tar.gz
```

The docker deployment demonstrated in this repository also uses the same `launcher.js` script introduced above to automatically initialize meteor environment variables from their Cloud Foundry counterparts. With that out of the way, let's build and push the docker image

```bash
docker build -t meshcloud/meteor-cf-example .
docker push meshcloud/meteor-cf-example:latest
```

With the container available in a docker registry, we can push it to Cloud Foundry by specifying the [docker image in the manifest](https://docs.cloudfoundry.org/devguide/deploy-apps/manifest.html#docker): 

```yml
---
applications:
- name: try-meteor-app-docker
  memory: 512M
  instances: 1
  docker:
    image: meshcloud/meteor-cf-example
  services:
    - meteor-mongo
```

Now all that's left to do is a simple `cf push` and your app should be up and running on Cloud Foundry in no time. 

```bash
iDevBook01:deploy-docker jr (master *) $ cf push
Pushing from manifest to org meshcloud-demo / space aproject as c9f7d64c-404d-4b29-b719-b2359f6c8157...
Using manifest file /Users/jr/dev/demo/meteor/deploy-docker/manifest.yml
Getting app info...
Updating app with these attributes...
  name:                try-meteor-app-docker
  docker image:        meshcloud/meteor-cf-example
  command:             node launcher.js
  disk quota:          1G
  health check type:   port
  instances:           1
  memory:              512M
  stack:               cflinuxfs2
  services:
    meteor-mongo
  routes:
    try-meteor-app-docker.cf.eu-de-darz.msh.host

Updating app try-meteor-app-docker...
Mapping routes...

Stopping app...

Waiting for app to start...

name:              try-meteor-app-docker
requested state:   started
instances:         1/1
usage:             512M x 1 instances
routes:            try-meteor-app-docker.cf.eu-de-darz.msh.host
last uploaded:     Mon 12 Feb 10:25:29 CET 2018
stack:             cflinuxfs2
docker image:      meshcloud/meteor-cf-example
start command:     node launcher.js

     state     since                  cpu    memory          disk         details
#0   running   2018-02-12T10:12:23Z   0.1%   38.2M of 512M   1.3M of 1G   

iDevBook01:deploy-docker jr (master *) $ 
```