# openshift-intern-workshop

### Install OpenShift client tools

Install via DNF

```
dnf install origin-clients
```

or donwload from Github - https://github.com/openshift/origin/releases/tag/v3.11.0

### Get code

Fork this repository (https://github.com/vpavlin/openshift-intern-workshop/fork) and clone it to your machine

### Login to the OpenShift cluster

You can either use your username and password

```
oc login https://<CLUSTER_URL>:8443 -u <USERNAME>
```

or you can use authentication token. To get a token, go to OpenShift Console, login, click your username in the top right corner and click *Copy Login Command*.

The whole command will be copied and you only need to paste it in the terminal and run it.

### Install Python ecosystem specific packaging tools

To be able to install pre-commit package bellow, you will need a tool `pip`

```
dnf install python-pip
```

### Configure pre-commit

[Pre-commit](https://pre-commit.com) as a framework for managing and maintaining multi-language pre-commit hooks. The tool is very useful when you are submitting changes to your or upstream repositories as it'll help you to keep the code clean and prevent many potential issues which might be noticed during review or not at all.

To install pre-commit, run

```
pip install --user pre-commit
```

Take a look at the file `.pre-commit-config.yaml` which contains configuration for pre-commit used in this repository.

To install the pre-commit hooks to your git repository run

```
pre-commit install
```

Now every time you do a git commit a set of checks will be run against your changes and errors will be reported and some even fixed automatically.

### Deploy the application

We need to deploy our application now. To verify you are successfully logged into OpenShift, you can run a following command

```
oc project
```

If there is no project you could use, create a new one:

```
oc new-project <YOUR_NAME>-intern-workshop
```

OpenShift/Kubernetes use JSON and YAML format to describe the deployment artifacts. You can find all the artifacts in YAML format in `openshift/` directory

To deploy whole application, you can pass a directory to oc **apply** command:

```
oc apply -f openshift/
```

Go to OpenShift Console and you should see a **build** running. Wait for the build to finish and for the deployment to proceed. Then try to access the **route** URL

### Fix the port

If you tried to access the application URL you were probably presented with *Application is not available* error. This could happen from various reasons, but the first thing we can check is whether our hostname and port in Flask application are configured properly.

Look at the last line in `app.py` file - you'll see we load the port from environment variable or use port `5000` as a default. Also look at the deployment config in `openshift/app.deploymentconfg.yaml` and focus on 2 things:

* a field `containerPort` in containers section
* an environment variabe `PORT`

As you can see these do not match. As changing `containerPort` would also require changing the **service** ports definition, we'll go for the envirnment variable and change it to `8080` to match the exposed port. We need to update the deployment config in the cluster. We can do this by running

```
oc apply -f openshift/app.deploymentconfig.yaml
```

Once the application is redeployed, you should get response like this:

```
{"msg":"Forbidden","status":403}
```

### Service account & roles

To get through the authentication you need to provide the URL with a secret in a query parameter, so try to add

```
?secret=secret
```

to the application address.

**Internal server error** - that does not look good - what have we missed? Let's investigate logs again - go to OpenShift Console > Applications > Pods and view the pod logs.

You will see something like the following error among the log messages:

```
HTTP response body: b'{"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"pods is forbidden: User \\"system:serviceaccount:vpavlin-os-ws:default\\" cannot list pods in the namespace \\"vpavlin-os-ws\\": no RBAC policy matched","reason":"Forbidden","details":{"kind":"pods"},"code":403}\n'
```

This is becuase our application is trying to access OpenShift API without proper authorization (you can see authentication is ok - OpenShift recognized the provided service account, but it does not have the correct rights to access resources it is trying to access).

We need to add a correct **role** to our service account. You can see the error mentions service account (or SA) **default**- The best practice would be to create a separate SA for this use case, but let's just change the default one for now.

We will need to add a **view** role for the SA and we can do this by running the following command

```
oc adm policy add-role-to-user view -z default
```

If the command succeeded, you will be able to reload the app URL and get back a JSON response.

### Change default secret

Let's look at **secrets** now. Secrets and Config Maps are resources used for app configuration. Our application uses one secret as well - `openshift/app.secret.yaml`. Take a look at it.

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

As secrets and config maps are mainly used in environment variables (which cannot be changed dynamically at runtime from outside of the container), we need to re-deploy our application to pick up new secret.

```
oc rollout latest openshift-intern-workshop
```

Once the deployment is finished, you will need to provide a new secret in the URL to be able to access the application.

### Health checks

OpenShift/Kubernetes come with a feature called health checks. There are 2 kinds of health checks - readiness and liveness probe. These are very important for lifecycle management of your application.

Readiness probe is a check which verifies if your application is fully up and running and ready to accept requests.

Liveness probe is used after readiness probe succeeds to repeatedly verify application is still fully up and alive.

If something fails in the container without actually failing the whole container or pod, you app may end up in inconsistent state. The simplest way to get to a consistent state is to restart the application - if readiness or liveness probe fail, OpenShift will restart the pod to get to a consistent state.

#### Readiness Probe

Go to *OpenShift Console > Applications > Deployments > openshift-intern-workshop*. Then select *Actions > Edit Health Checks* on the right. Click *Add Readiness Probe*.

Look at the `app.py` - which of the API calls would you use for this (hint: health)? Add the path (including leading forward slash) to the Path field. You can leave the rest as is. Click Save.

Once your application is redeployed it will verify it is started up properly before OpenShift sends any traffic to the pod.

#### Liveness Probe

Let's add the Liveness Probe the hard way:). Go back to *OpenShift Console > Applications > Deployments > openshift-intern-workshop* again, but this time click *Actions > Edit YAML*.

Find the `readinessProbe` section, duplicate it and change the name to `livenessProbe`. Make sure the indentation is the same and you pasted the whole block right below the readiness probe block.

Click Save and wait for a new deployment. If you go to Applications > Pods and open the latest openshift-intern-workshop pod, you should see both readiness and liveness probes mentioned on the right

```
Readiness Probe: GET /health on port 8080 (HTTP) 1s timeout
Liveness Probe: GET /health on port 8080 (HTTP) 1s timeout
```

### Resource limits

Another important feature of OpenShift and Kubernetes is to make sure your application has enough resources, but at the same time does not consume more than an administrator allows. For this it exposes resource **requests** and **limits** in the pod specification.

#### Limits

First, let's set limits. Limits make sure that your application does not consume too many resources and OpenShift will kill the container if it does.

We will do this similarly to how we set readiness probe - via the console. Go to *Console > Applications > Deployments > openshift-intern-workshop > Action > Edit Resource Limits*.

You can see that there are some values preconfigured as defaults. Let's limit our application to `200` milicores and `600` megabytes of RAM. Fill those values in *Limit* fields and click *Save*.

#### Requests

Requests make sure your application is deployed in a way and on a node which provides enough free resources.

We will configure requests similarly to how we configured the liveness probe - go to *OpenShift Console > Applications > Deployments > openshift-intern-workshop > Actions > Edit YAML*. Locate `limits` section, duplicate it (make sure indentation is correct) and change it's name to `requests`. No change the values - set memory to `300Mi` and cpu to `100m` and click *Save*

You should see this in the pod details view

```
CPU: 100 millicores to 200 millicores
Memory: 300 MiB to 600 MiB
```

### Scaling

Now that we have set our memory and CPU requests, we can try to scale our application.

As we are loading the information about running pods directly from OpenShift API, we can try to change number of pods and see if the API response changes

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

Our application uses [Source-To-Image](https://github.com/openshift/source-to-image) (or S2I). S2I is a smart tool which makes it easy to build application container images. Look at the `openshift/app.buildconfig.yaml` to see how the S2I strategy is configured.

You can notice we need to provide 3 pieces of information

* Source image
* Source repository
* Output image

Source image is a contiainer image which was designed for working with S2I - apart from other features it contains `assemble` and `run` scripts - you can see and example here: https://github.com/sclorg/s2i-python-container/blob/master/3.6/s2i/bin/assemble - which are used during build and start of the container.

Source repository is a git repository containing application in a language matching the one of a source container image, so that the tools in the source image know how to install the application.

Output image is a name of an image stream where the resulting container image will be pushed.

To be able to successfully build from your own repository, do not forget to change the *build config* source repository to your own!

#### Building from local directory

When you develop your code you will need to rebuild the container image for your application. Our application was originally built and deployed from a git repository. To be able to quickly rebuild your knew changes you might want to skip the step of pushing your code to a repository and then kicking off the build.

To do that, you can use (make sure you are in the root of the repository)

```
oc start-build openshift-intern-workshop --from-dir=. -F
```

Start build command will start new build in OpenShift and `--from-dir` will collect contents of a given directory, compress it and send it to OpenShift as a context directory for the new build. Parameter `-F` fill redirect logs from the build to the terminal, so that you can easily look at how the build progresses.

Once the build is finished, OpenShift will automatically redeploy our application - this happens based on **triggers** defined in `openshift/app.deploymentconfig.yaml`

#### Setting up webhooks

Webhooks are a powerful automation feature provided by both OpenShift and Github. OpenShift will act as a reciever of a webhook request and Github will produce webhook calls when we push to the repository.

First go to OpenShift Console > Builds > Builds > openshift-intern-workshop > Configuration and copy the *Github Webhook URL*.

Next go to your Github wokrshop repository and click Setting > Webhooks > Add webhook. Paste the copied URL in *Payload URL*, change *Content type* to `application/json` and disable *SSL verification* and confirm by clicking *Add webhook*.

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

To verify the change, go to OpenShift Console > Builds > Builds > openshift-intern-workshop > Configuration and check the value of *Source Ref:*

You can now commit and push your changes

```
git commit -a -m "Add service list to API"
git push --set-upstream origin feature/services
```
Look at the *OpenShift Console > Builds > Builds > openshift-intern-workshop > History* - you will see a new build running, if your webhook is configured correctly. Wait for the build and following deployment to finish and reload your application - you should see a service listed there as well now.

### Adding persistent volumes

Sometimes an application needs some persisentcy. The most classic example are databases - without a persistent volume all the data you store would be lost on container restart - and restarts happen a lot in a distributed cloud environment.

To simulate this situation, we have an endpoint in our app which stores a value in a file. First get the route of the app and store it in environment variable

```
APP_URL=$(oc get route openshift-intern-workshop -o jsonpath='{.spec.host}')
```

Next try to query the `/iam` endpoint

```
curl $APP_URL/iam
```

You will see a message: `Could not find the 'iam' file`

We need to set the value first by doing a POST request to the enpoint

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

As you can see, the value is gone. So let's make sure it gets properly persisted next time - let's add a **persisten volume** to our application. OpenShift uses something called **dynamic provisioning** to generate persistent volume based on **persistent volume claims** (or PVCs). Our task is only to create a PVC artifact and attach it to the pod and OpenShift will handle the rest.

Ideally you would do this by adding another YAML files to your git repository, but for the sake of simplicity, let's do it manually form the OpenShift Console. Go to the *console > Applications > Deployments > openshift-intern-workshop > Actions > Add storage*.

Click *Create storage*. Give your new PVC a name and size (e.g. 1 GB). Click Create. Then provide a mount path - if you look into `app.py` file, you'll notice that the value submitted to the `/iam` endpoint is stored in a file `./iam`. The full path to the file is `/opt/app-root/src/iam`. As the `/opt/app-root/src` directory contains our application, we will want to persist the file in a subdirectory. For that set the *Mount Path* to

```
/opt/app-root/src/data
```

and click *Add*.

We need to change the path in the source code as well - edit the `app.py` file and set the `IAM_FILE` value to `/opt/app-root/src/data/iam` - the line will now look like this:

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
