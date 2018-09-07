# Kubeless on Azure Kubernetes Service

This workshop walks through setting up [Kubeless](https://kubeless.io) on an
Azure Kubernetes Service cluster and deploying some example functions to get
familiar with Kubeless.

## Prerequisites

- Azure account with the ability to create [Kubernetes
  clusters](https://azure.microsoft.com/en-us/services/kubernetes-service/)
- [Azure
  CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
- [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## Setting up your Kubernetes cluster on Azure

Follow [this
walkthrough](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal)
to setup your Azure Kubernetes Service cluster and connect to it from your local
machine using `kubectl`.

## Installing Kubeless

Head over to the [Kubeless releases
page](https://github.com/kubeless/kubeless/releases) to download the latest
binary for your operating system. For example, for macOS:

```
curl -LO https://github.com/kubeless/kubeless/releases/download/v1.0.0-alpha.8/kubeless_darwin-amd64.zip
unzip kubeless_darwin-amd64.zip
chmod +x bundles/kubeless_darwin-amd64/kubeless
mv bundles/kubeless_darwin-amd64/kubeless /usr/local/bin/kubeless
```

One installed correctly, you should be able to run `kubeless version`, for
example:

```
$ kubeless version
Kubeless version: v1.0.0-alpha.8
```

To install the server-side components of Kubeless on your Kubernetes cluster,
copy and run the instructions in the release notes for the version you are
installing. For example, for version v1.0.0-alpha.8 in an RBAC-enabled cluster:

```
kubectl create ns kubeless
kubectl create -f https://github.com/kubeless/kubeless/releases/download/v1.0.0-alpha.8/kubeless-v1.0.0-alpha.8.yaml 
```

Once the server-side components have successfully been installed, you should be
able to run the `kubeless get-server-config` command:

```
$ kubeless get-server-config
INFO[0000] Current Server Config:
INFO[0000] Supported Runtimes are: python2.7, python3.4, python3.6, nodejs6, nodejs8, nodejs_distroless8, ruby2.4, php7.2, go1.10, dotnetcore2.0, java1.8, ballerina0.981.0, jvm1.8
```

## Discover the Kubeless API

With Kubeless now installed in your cluster, you'll notice that there is a new
`functions` API resource available to you:

```
$ kubectl get functions
No resources found.
```

We haven't created any functions yet, so this will come back empty. As we create
functions later in this walkthrough, we'll see this list will become populated.

## Clone this repository

The easiest way to use the examples in this repository is to clone it:

```
git clone https://github.com/prydonius/kubeless-on-azure-walkthrough-ossvc
cd kubeless-on-azure-walkthrough-ossvc
```

## Deploying our first function

The first function we will deploy is a simple one-line function that will return
the text "hello world!" in Node.js:

```
module.exports = {
  foo: function (event, context) {
    return 'hello world!';
  }
}
```

Run the following command to deploy the function:

```
kubeless function deploy helloget --runtime nodejs6 --handler helloget.foo --from-file helloget.js
```

We can see that creating this function has resulted in the creation of a Pod to
run our small bit of code:

```
$ kubectl get functions
NAME       AGE
helloget   1m

$ kubectl get pods
NAME                        READY     STATUS            RESTARTS   AGE
helloget-675b75d66d-nt68m   0/1       PodInitializing   0          5s
```

To run the function, we use the `kubeless function call` command:

```
$ kubeless function call helloget
hello world!
```

## Deploying functions declaratively using Kubernetes manifest and kubectl

One way to manage resources in your Kubernetes cluster is by writing declarative
definitions in YAML or JSON that describe the desired state of your cluster.
Kubeless fits into this workflow as it extends the Kubernetes API. To
demonstrate this, we can deploy the same function in the previous step using a
Kubernetes manifest:

```
apiVersion: kubeless.io/v1beta1
kind: Function
metadata:
  name: hello
spec:
  handler: handler.hello
  runtime: nodejs6
  function: |
      module.exports = {
        hello: function(event, context) {
          return 'Hello World directly using Kubernetes!'
        }
      }
```

This function is mostly the same as the previous one, but changes the message
returned to "Hello World directly using Kubernetes!".

To deploy this function using kubectl:

```
kubectl apply -f function.yaml
```

Once this function is running, we can try calling it:

```
$ kubeless function call hello
Hello World directly using Kubernetes!
```

## Reading data passed to a function

Typically, you would want to pass around some data along with your function call
to do some processing. With Kubeless, this is pretty easy. Your functions accept
and `event` and `context` argument that give it more information about the
incoming request.

To build a basic echo service using Kubeless functions, we can deploy the
following function:

```
module.exports = {
  handler: (event, context) => {
    console.log(event);
    return event.data;
  },
};
```

Deploy `hellowithdata.js` using kubeless:

```
kubeless function deploy hellowithdata --runtime nodejs6 --handler hellowithdata.handler --from-file hellowithdata.js
```

Once this function is up and running, we can try calling it with some data:

```
$ kubeless function call hellowithdata --data '{"hello": "world"}'
{"hello": "world"}
```

## Writing functions with language dependencies

When writing real-world functions, you might find you need to import some dependencies from the language's package manager. Kubeless supports installing dependencies for the core runtimes it supports.

As an example, we can create a function that uses the Node.js _lodash_ package:

```
'use strict';

const _ = require('lodash');

module.exports = {
    handler: (event, context) => {
        _.assign(event.data, {date: new Date().toTimeString()})
        return JSON.stringify(event.data);
    },
};
```

This, accompanied with the following `package.json` file to define the dependency and version, will instruct Kubeless to fetch this package before running the function:

```
{
    "name": "hellowithdeps",
    "version": "0.0.1",
    "dependencies": {
        "lodash": "3.10.1"
    }
}
```

To deploy this, we'll need to specify the `--dependencies` flag and point it to our `package.json` file:

```
kubeless function deploy hellowithdeps --runtime nodejs6 --handler hellowithdeps.handler --from-file hellowithdeps.js --dependencies package.json
```

Once the dependencies have been installed and the function is running, we can test it by calling it:

```
$ kubeless function call hellowithdeps --data '{"hello": "world"}'
{"hello": "world","date":"00:43:31 GMT+0000 (UTC)"}
```

## Cleanup

To remove the example functions from your cluster:

```
kubectl delete functions --all
```

To remove Kubeless from your cluster:

```
kubectl delete -f https://github.com/kubeless/kubeless/releases/download/v1.0.0-alpha.8/kubeless-v1.0.0-alpha.8.yaml 
```

## Other resources

Checkout the [Kubeless documentation](https://kubeless.io/docs) to learn more about Kubeless and what you can do with it!
