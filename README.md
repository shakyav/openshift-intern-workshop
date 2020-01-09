# openshift-intern-workshop

### Install OpenShift client tools

Download the oc client binary: https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.2.7/

Extract the client, for example:
```
tar -xvf openshift-client-linux-4.2.7.tar.gz
```

You can now move the client into your `$PATH`, for example:

`mv oc /home/$USER/bin` (you may need to create the directory)

or you can use the client with a relative path: `./oc`

For the rest of the tutorial we will assume the `oc` client is in your path.

### Get code

Fork this repository (https://github.com/patrickdillon/openshift-intern-workshop/fork) and clone it to your machine

### Login to the OpenShift cluster

We will use a 4.2 OpenShift cluster from the Red Hat Product Demo System for this workshop.

You can either use your username and password:
```
oc login https://<CLUSTER_URL>:6443 -u <USERNAME>
```

or you can use authentication token. To get a token, go to OpenShift Console, login, click your username in the top right corner and click Copy Login Command.

The whole command will be copied and you only need to paste it in the terminal and run it.


### Install Python ecosystem specific packaging tools

To be able to install pre-commit package bellow, you will need a tool `pip`

```
dnf install python-pip
```

### Configure pre-commit

[Pre-commit](https://pre-commit.com) is a framework for managing and maintaining multi-language pre-commit hooks. The tool is useful when you are submitting changes as it helps you keep code clean and prevent potential issues which might be noticed during review or not at all.

To install pre-commit, run:

```
pip install --user pre-commit
```

Take a look at the file `.pre-commit-config.yaml` which contains configuration for pre-commit used in this repository.

To install the pre-commit hooks to your git repository run

```
pre-commit install
```

Now, when you `git commit` a set of checks will be run against your changes and errors will be reported or even fixed automatically.

### Deploy the application

We need to deploy our application now. To verify you are successfully logged into OpenShift, you can run the following command:

```
oc project
```

If there is no project you could use, create a new one:

```
oc new-project <YOUR_NAME>-intern-workshop
```

OpenShift/Kubernetes uses JSON and YAML format to describe the deployment artifacts. You can find all the manifests in YAML format in `openshift/` directory

To deploy the whole application, you can pass a directory to the oc **apply** command:

```
oc apply -f openshift/
```

Go to the OpenShift Console and you should see a **build** running. Wait for the build to finish and for the deployment to proceed. Then try to access the **route** URL. You can get the URL under `HOST/PORT` by:

```
$ oc get route
```

### Fix the port

If you tried to access the application URL you were probably presented with *Application is not available* error. This could happen for various reasons, but the first thing we can check is whether our hostname and port in the Flask application are configured properly.

Look at the last line in [app.py](./app.py) file - you'll see we load the port from environment variable or use port `5000` as a default. Also look at the deployment config in [openshift/app.deploymentconfig.yaml](openshift/app.deploymentconfig.yaml) and focus on 2 things:

* a field `containerPort` in containers section
* an environment variabe `PORT`

As you can see these do not match. As changing `containerPort` would also require changing the **service** ports definition, we'll go for the envirnment variable.

Go ahead and change the `5000` to `8080` to match the exposed port in the `openshift/app.deploymentconfig.yaml` file. We will then need to update the deployment config in the cluster by running

```
oc apply -f openshift/app.deploymentconfig.yaml
```

Once the application is redeployed, try to access the **route** URL again. You should get response like this:

```
{"msg":"Forbidden","status":403}
```

### Service account & roles

To get through the authentication you need to provide the URL with a secret in a query parameter, so try to append the following to your **route** URL:

```
?secret=secret
```

**Internal server error** - that does not look good - what have we missed? Let's investigate logs again - go to OpenShift Console > Workloads > Pods and view the pod logs.

You will see something like the following error among the log messages:

```
HTTP response body: b'{"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"pods is forbidden: User \\"system:serviceaccount:vpavlin-os-ws:default\\" cannot list pods in the namespace \\"vpavlin-os-ws\\": no RBAC policy matched","reason":"Forbidden","details":{"kind":"pods"},"code":403}\n'
```

Our application is trying to access OpenShift API without proper authorization (you can see authentication is ok - OpenShift recognized the provided service account, but it does not have the correct rights to access resources it is trying to access).

We need to add a correct **role** to our service account. You can see the error mentions service account (or SA) **default**- The best practice would be to create a separate SA for this use case, but let's just change the default one for now.

We will need to add a **view** role for the SA and we can do this by running the following command

```
oc adm policy add-role-to-user view -z default
```

If the command succeeded, you will be able to reload the app URL and get back a JSON response such as:
```json
{"me":"openshift-intern-workshop-x-xxxxx","pods":["openshift-intern-workshop-x-xxxxx"],"routes":["openshift-intern-workshop"],"services":[]}
```

### Change default secret

Let's look at **secrets** now. Secrets and Config Maps are resources used for app configuration. Our application uses one secret as well - [openshift/app.secret.yaml](openshift/app.secret.yaml). Take a look at it.

The interesting part is in section `data`, but you cannot easily read it. The secret is obfuscated by **base64** encoding to make it slightly harder to leak it by showing to someone. If we want to read it, we can copy the value and pass it through the base64 decoder

```
echo "c2VjcmV0" | base64 -d
```

So the actual value is `secret`. Let's change it now! Pick a new secret and use it in following command

```
echo -n "<MYNEWSECRET>" | base64
```

Now copy the value and edit the secret in OpenShift

```
oc edit secret openshift-intern-workshop
```

Look at the file `openshift/app.deploymentconfig.yaml` and try to find how the secret is used there.

As secrets and config maps are mainly used in environment variables (which cannot be changed dynamically at runtime from outside of the container), we need to re-deploy our application to pick up the new secret.

```bash
oc rollout latest openshift-intern-workshop
```

Once the deployment is finished, you will need to provide the new secret in the URL to be able to access the application by attaching `?secret=<MYNEWSECRET>` to the **route** URL.

#### Health Checks: Liveness & Readiness Probe

Health checks are an important tool for the lifecycle management of an application. Readiness and liveness probes can be used to determine that a container is functioning properly.

A readiness probe is a check which verifies if your application is fully up and running and ready to accept requests. When readiness probes fail the container will not be assigned an ip address.

Liveness probes are subsequently used to repeatedly verify the application is up. If a liveness probe fails, the container will be restarted (based on the restart policy).

For our application, look at [app.py](./app.py) and you will see a path called "health" which we can use in our health check.

Let's create a liveness probe.

Go *OpenShift Console > Workloads > Deployment Configs > openshift-intern-workshop* and click *YAML*.


You will see a section that looks like this:
```yaml
[...]
containers:
  - resources: {}
    terminationMessagePath: /dev/termination-log
    name: openshift-intern-workshop
    env:
      - name: PORT
        value: '8080'
[...]
```

We need to add a `livenessProbe` to our `openshift-intern-workshop` container. Please note that the ordering of the fields below are not important but the `livenessProbe` section has to indented correctly.

```yaml
[...]
containers:
  - resources: {}
    terminationMessagePath: /dev/termination-log
    name: openshift-intern-workshop
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 15
      timeoutSeconds: 1
    env:
      - name: PORT
        value: '8080'
[...]
```

Click Save and wait for a new deployment. If you click *Pods*, open the latest openshift-intern-workshop pod and select the `openshift-intern-workshop` container, you should see the liveness probe registered on the left hand side.

Readiness probes can be created in the same way by replacing `livenessProbe` with `readinessProbe`.

### Resource limits

Another important feature of OpenShift and Kubernetes is to make sure your application has enough resources, but at the same time does not consume more than an administrator allows. For this, we use resource **requests** and **limits** in the pod specification.

#### Limits

Limits ensure that your application does not consume too many resources and the OpenShift controller will kill the container if it does.

We have already edited Kubernetes objects using two methods: `oc edit` and editing YAML through the console. We will change the limits of our application by editing the DeploymentConfig but let's do it in yet another way, the `oc patch` command:

```bash
oc patch deploymentconfig openshift-intern-workshop -p '{"spec":{"template":{"spec":{"containers":[{"name":"openshift-intern-workshop", "resources": {"limits": {"memory": "600Mi", "cpu":"200m"}}}]}}}}'
```

This limits our application to `200` millicores and `600` megabytes of RAM.


#### Requests

Requests make sure your application has sufficient resources.

```bash
oc patch dc openshift-intern-workshop -p '{"spec":{"template":{"spec":{"containers":[{"name":"openshift-intern-workshop", "resources": {"requests": {"memory": "300Mi", "cpu":"100m"}}}]}}}}'
```

Our application now has between 300-600 megabytes of memory and 100-200 millicores of CPU. With `oc describe pod` we can see that our pod now has a [Quality of Service](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/) of burstable:
```
QoS Class:       Burstable
```

### Scaling

Now that we have set our memory and CPU requests, we can attempt to scale our application.

As we are loading the information about running pods directly from OpenShift API, we can change the number of pods and see if the API response changes

```
oc scale --replicas=5 dc openshift-intern-workshop
```

Go to your app URL and reload it couple times - you will see the list of pods is longer now.

Now that the application is running in multiple replicas, we would like to see some load balancing. Look at the name in the `me` field - take that value and use it in next command to kill that container

```
oc delete pod <CONTENT_OF_ME_FIELD>
```

Now if you reload your browser tab, you will see the value in `me` field changed - i.e. the traffic goes to a different pod as the original one no longer exists.

Let's scale our deployment down again to make sure we don't disrupt the cluster and that redeployment does not take unnecessarily long time

```
oc scale --replicas=1 dc openshift-intern-workshop
```

### Changing the code

Our application uses [Source-To-Image](https://github.com/openshift/source-to-image) (or S2I). S2I is a smart tool which makes it easy to build application container images. Look at the [openshift/app.buildconfig.yaml](openshift/app.buildconfig.yaml) to see how the S2I strategy is configured.

You can notice we need to provide 3 pieces of information

* Source image
* Source repository
* Output image

Source image is a container image which was designed for working with S2I - apart from other features it contains `assemble` and `run` scripts - you can see an example here: https://github.com/sclorg/s2i-python-container/blob/master/3.6/s2i/bin/assemble - which are used during build and start of the container.

Source repository is a git repository containing application in a language matching the one of a source container image, so that the tools in the source image know how to install the application.

Output image is a name of an image stream where the resulting container image will be pushed.

To be able to successfully build from your own repository, do not forget to change the *build config* source repository to your own!

#### Building from local directory

When you develop your code you will need to rebuild the container image for your application. Our application was originally built and deployed from a git repository. To be able to quickly rebuild your new changes you might want to skip the step of pushing your code to a repository and then kicking off the build.

To do that, you can use (make sure you are in the root of the repository)

```
oc start-build openshift-intern-workshop --from-dir=. -F
```

Start build command will start new build in OpenShift and `--from-dir` will collect contents of a given directory, compress it and send it to OpenShift as a context directory for the new build. Parameter `-F` fill redirect logs from the build to the terminal, so that you can easily look at how the build progresses.

Once the build is finished, OpenShift will automatically redeploy our application - this happens based on **triggers** defined in [openshift/app.deploymentconfig.yaml](openshift/app.deploymentconfig.yaml)

#### Setting up webhooks

Webhooks are a powerful automation feature provided by both OpenShift and Github. OpenShift will act as a receiver of a webhook request and Github will produce webhook calls when we push to the repository.

First go to OpenShift Console > Builds > Build Configs > openshift-intern-workshop > Configuration and *Copy with Secret* the *Github Webhook URL*.

Next go to your Github workshop repository and click Setting > Webhooks > Add webhook. Paste the copied URL in *Payload URL*, change *Content type* to `application/json` and disable *SSL verification* and confirm by clicking *Add webhook*.

### Add services to response

Create a new branch in your repository

```
git checkout -b feature/services
```

Look at the code in `workshop/openshift_info.py` and try to implement a method similar to `get_pods`, but instead of a list of Pod names, make it to return list of Service names

<details><summary>Solution</summary>
<p>

File `workshop/openshift_info.py`

```python
    def get_services(self):
        services_api = self.oapi_client.resources.get(
                            kind='Service',
                            api_version='v1')
        service_list = services_api.get(namespace=self.namespace)
        return self._get_names(service_list)
```

</p>
</details>

#### Configure a build config to pull from a branch

You are now making changes to your code in a new branch, but the build config is pulling from `master`. To make sure you build from the latest changes, we will need to add `ref` to our build config.

```
oc edit bc openshift-intern-workshop
```

Find a section `source` and in there find `uri`, which should point to your repository fork (Make sure it has correct value too!). Add the following line right under the `uri` field and make sure the indentation is the same

```
ref: feature/services
```

To verify the change, go to OpenShift Console > Builds > Build Configs > openshift-intern-workshop > Configuration and check the value of *Source Ref:*

You can now commit and push your changes

```
git commit -a -m "Add service list to API"
git push --set-upstream origin feature/services
```
Look at the *OpenShift Console > Builds > Builds > openshift-intern-workshop > History* - you will see a new build running, if your webhook is configured correctly. Wait for the build and following deployment to finish and reload your application - you should see a service listed there as well now.

### Adding persistent volumes

Sometimes an application needs some persistency. The most classic example are databases - without a persistent volume all the data you store would be lost on container restart - and restarts happen a lot in a distributed cloud environment.

To simulate this situation, we have an endpoint in our app which stores a value in a file. First get the route of the app and store it in an environment variable

```
APP_URL=$(oc get route openshift-intern-workshop -o jsonpath='{.spec.host}')
```

Next try to query the `/iam` endpoint

```
curl $APP_URL/iam
```

You will see a message: `Could not find the 'iam' file`

We need to set the value first by doing a POST request to the endpoint

```
curl -X POST $APP_URL/iam/<YOUR_NAME_HERE>
```

If this succeeded, you should get your name back when you do the GET request on the endpoint again

```
curl $APP_URL/iam
```

Now let's delete/restart the pod and see that the value is gone

```
POD=$(oc get pods | grep Running | awk '{print $1}')
oc delete pod $POD
```

Hit the endpoint again when the pod comes back up

```
curl $APP_URL/iam
```

As you can see, the value is gone. So let's make sure it gets properly persisted next time - let's add a **persistent volume** to our application. OpenShift uses something called **dynamic provisioning** to generate persistent volume based on **persistent volume claims** (or PVCs). Our task is only to create a PVC artifact and attach it to the pod and OpenShift will handle the rest.

Ideally you would do this by adding another YAML files to your git repository, but for the sake of simplicity, let's do it manually form the OpenShift Console. Go to the *console > Workloads > Deployment Configs > openshift-intern-workshop > Actions > Add storage*.

Click *Create storage*. Give your new PVC a name and size (e.g. 1 GB). Click Create. Then provide a mount path - if you look into `app.py` file, you'll notice that the value submitted to the `/iam` endpoint is stored in a file `./iam`. The full path to the file is `/opt/app-root/src/iam`. As the `/opt/app-root/src` directory contains our application, we will want to persist the file in a subdirectory. For that set the *Mount Path* to

```
/opt/app-root/src/data
```

and click *Add*.

We need to change the path in the source code as well - edit the [app.py](./app.py) file and set the `IAM_FILE` value to `/opt/app-root/src/data/iam` - the line will now look like this:

```
IAM_FILE = "/opt/app-root/src/data/iam"
```

To get the change in we need to rebuild the container image - you can push the change to your repository, or use the build from a local dir - you have tried both before.

```
 oc start-build openshift-intern-workshop --from-dir=. -F
```

Once the image is rebuilt and the application redeployed, we can send the `POST` request again

```
curl -X POST $APP_URL/iam/<YOUR_NAME_HERE>
```

Then check the value is set properly

```
curl $APP_URL/iam
```

Delete (restart) the pod

```
POD=$(oc get pods | grep Running | awk '{print $1}')
oc delete pod $POD
```

and when it comes back up, see that the value is still there

```
curl $APP_URL/iam
```

# Learn more

Want to learn more? Go to http://learn.openshift.com/ and try more free workshops.
