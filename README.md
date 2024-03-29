# FluxCD Gitops using image automation and image reflector controller

This is a documentation, designed to guide you through the practical implementation of a GitOps workflow using FluxCD. In this documentation, we'll explore how to leverage the power of image automation and the image reflector controller to streamline your development and deployment processes.

We will showcase this through a deployment pipeline in 2 env (staging and production) wherein using githubactions we will do the CI process of building and pushing the containers to registry , once pushed to registry, the image automation and image reflector controllers will identify the change and make an automatic update to kubernetes manifests and flux will automatically reconcile the changes in stage env. For Prod, to ensure control, changes are promoted to production through a Pull Request (PR) with manual approval.
![image](https://github.com/shashankpai/gitops-helloenv/assets/24639491/bbe898c1-ecc2-461e-ad53-bfdd8f80a10b)
![image](https://github.com/shashankpai/gitops-helloenv/assets/24639491/da68a611-aa6d-46b0-a2d8-5939808cd27a)

### Prerequisites

Ensure you have following things setup:

1. **Flux CLI** - which can be downloaded from the [official docs](https://fluxcd.io/docs/cmd/)
2. **Kubernetes cluster** -  for this demo, we will use minikube.
3. GitHub account - which has GitHub Actions enabled.

### Environment setup details

1. **Hello Env Flask application**: A simple application displaying the environment name and release number.
2. **Two environments**: Staging and production, housed in separate namespaces.
3. **Continuous deployment**: Source code changes trigger automated build and deployment to the staging environment.
4. **Release tagging**: Tagging a release initiates automatic build processes and creates a pull request for production deployment.


#### Git repositories

Let's take a look at two Git repositories we will be using.

1. **Application repository** - [https://github.com/infracloudio/flux-helloenv-app](https://github.com/infracloudio/flux-helloenv-app)  - code and Kustomization.
   1. **Kustomization for environment separation**: Utilizing Kustomize to segregate staging and production environments. 
   2. **Structured repository**: The repository combines source code and Kubernetes manifests for a unified demo setup. There are many possible ways to [structure your git repositories.](https://fluxcd.io/flux/guides/repository-structure/)
   3. **CI and automation**: Leveraging GitHub Actions for [continuous integration](/ci-cd-consulting/) and automation.
2. **Management Repository**-  [https://github.com/infracloudio/flux-gitops-helloenv](https://github.com/infracloudio/flux-gitops-helloenv) - GitOps manifests.

### Flux bootstrap configuration

Beginning with an empty cluster, our initial task is to [bootstrap Flux itself](https://fluxcd.io/flux/cmd/flux_bootstrap/). Flux serves as the foundation upon which we'll bootstrap all other components.

In the following sections, we'll take a detailed look at each file within the apps directory, one by one. 

```sh
├── apps
│   ├── git-repo.yaml
│   ├── image-auto-prod.yaml
│   ├── image-auto-staging.yaml
│   ├── image-policy-prod.yaml
│   ├── image-policy-staging.yaml
│   ├── image-repo.yaml
│   ├── kustomization-prod.yaml
│   └── kustomization-staging.yaml
```
 
#### git-repo.yaml

This instructs Flux on how to interact with the Git repository where your application's source code resides. It contains the following configuration:

```yaml
  ref:
    branch: main
  secretRef:
    name: ssh-credentials
  url: ssh://git@github.com/infracloudio/flux-helloenv-app
```

`ref:` Specifies the Git branch to watch for changes, in this case, main.  
`secretRef:` Refers to a Kubernetes Secret named ssh-credentials, which likely contains SSH keys for secure Git access.  
The `ssh-credentials` is a secret that needs to be created.  

`url:` Indicates the URL of the Git repository, Change this to `url: ssh://git@github.com/<your_github_username>/helloenv-app` once you fork it.  

#### image-repo.yaml

This file is responsible for scanning the Docker image registry and fetching image tags based on the defined policy. Here's the configuration:  

```yaml
image: docker.io/shapai/helloenv
interval: 1m0s
```

`image:` Specifies the Docker image repository (docker.io/shapai/helloenv) to scan for image tags. Change this to have your relevant Docker image repository, where the CI job will build and push the image. This same registry needs to be updated in your forked CI file, where the CI build will push images.  
`interval:` Sets the interval at which Flux will scan the image repository (every 1 minute in this case) and fetch image tags according to the defined policy.  
  
Next, we will go through the image policy files image-policy-staging.yaml and image-policy-prod.yaml  

#### image-policy-staging.yaml 

This file defines the image tagging policy for the staging environment. Here's the configuration:  

```yaml
  filterTags:
    extract: $ts
    pattern: ^main-[a-f0-9]+-(?P<ts>[0-9]+)
  imageRepositoryRef:
    name: helloenv
  policy:
    numerical:
      order: asc
```

`filterTags:` This section specifies how to filter image tags. It extracts the timestamp ($ts) from tags that match the specified pattern. Tags are filtered in ascending order based on this timestamp, ensuring that Flux fetches the latest built image for the staging environment.  

#### image-policy-prod.yaml 

This file defines the image tagging policy for the production environment:  

```yaml
  imageRepositoryRef:
    name: helloenv
  policy:
    semver:
      range: '>=1.0.0'
```

`imageRepositoryRef:` Refers to the image repository named helloenv.  
`policy:` Defines a Semantic Versioning (SemVer) policy that specifies a range for acceptable image tags (in this case, any version greater than or equal to 1.0.0).  
These image policies are crucial in ensuring that Flux deploys the correct images to the respective environments (staging and production) based on the defined criteria.  

#### kustomization-prod.yaml and kustomization-staging.yaml

These YAML files define Kustomization resources for managing Kubernetes resources in both staging and production environments within the flux-system namespace. They are configured to synchronize with a specified Git repository, allowing for automated deployment and management of Kubernetes resources.
 

Now, let's go through the **ImageUpdateAutomation** files.

#### helloenv-staging.yaml

This YAML file configures an ImageUpdateAutomation object to update the image tags in the `./demo/kustomize/staging` directory of the helloenv-app Git repository. It will scan the repo every 1 minute (specified by the interval field).   

The `git` section specifies the branch to checkout and the commit message template. The sourceRef section specifies the Git repository containing the Kubernetes manifests to update.  

The `update` section specifies the path to the Kubernetes manifests to update and the strategy to use.

When this _ImageUpdateAutomation_ object is deployed, Flux will periodically check for new image updates. If it finds any new updates, it will update the image tags in the Kubernetes manifests and commit the changes to the Git repository.

{% raw %}
```yaml
spec:
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxbot@users.noreply.github.com
        name: fluxbot
      messageTemplate: '{{range .Updated.Images}}{{println .}}{{end}}'
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: helloenv-app
  update:
    path: ./demo/kustomize/staging
    strategy: Setters
```
{% endraw %}

#### helloenv-prod.yaml  

This YAML file is very similar to the previous one but has an additional push configuration. This means that after updating the image tags in the Git repository, Flux will commit and push the changes to the flux-image-update branch. This is done specially for prod setup since we are not going to push directly to the main branch for prod manifests.  

This is useful in our case, as this will help us create a PR to the main branch from the flux-image-update branch, which will then go through manual approval. The process of creating PR is handled by the CI section in GitHub Actions.

{% raw %}
```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: helloenv-prod
  namespace: flux-system
spec:
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxbot@users.noreply.github.com
        name: fluxbot
      messageTemplate: '{{range .Updated.Images}}{{println .}}{{end}}'
    push:
      branch: flux-image-update
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: helloenv-app
  update:
    path: ./demo/kustomize/prod
    strategy: Setters
```
{% endraw %} 

### Setting up Flux

Now that we've seen all the configurations, we can proceed to the bootstrap command.

Note the `--owner` and `--repository` switches here: we are explicitly looking for the `${GITHUB_USER}/gitops-helloenv` repo. Make sure you fork both repos under your user and follow on.

```sh
flux bootstrap github \ 
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=$GITHUB_USER \
  --repository=gitops-helloenv \
  --path=./clusters/my-cluster/ \
  --branch=main \
  --read-write-key \
  --personal --private=false
```

#### Understanding `flux bootstrap` command

Let's understand what the above command does.

To enable the Flux image automation feature, the extra components can be specified with the [`--components-extra` flag](https://fluxcd.io/flux/installation/configuration/optional-components/). We are enabling the image reflector and automation controllers.

`--path=./clusters/my-cluster/` specifies the path within the repository where the configuration files will be stored.  

`my-cluster` folder in the repository contains two files `infrastructure.yaml` and `apps.yaml`.  

- `infrastructure.yaml` defines a Kustomization resource called ingress-nginx. This Kustomization lives in the flux-system namespace, doesn't depend on anything, and has kustomize files at infrastructure/ingress-nginx.

- `apps.yaml`, which defines the `helloenv-app` application itself. Within this file, you'll find references to the apps directory. This directory is a central location containing Kustomizations and configuration settings for setting up the `helloenv-app` application in both the prod and staging environments. Additionally, it includes configurations for image automation for `helloenv` app in staging and prod env.

The process of bootstrapping everything may take some time. To monitor the progress and ensure everything is proceeding as expected, we can utilize the `--watch` switch.

```sh
flux get kustomizations --watch
```

Verify that all Flux pods are in running state by running get pods.
```sh
$ kubectl -n flux-system get pods

NAME                                           READY   STATUS    RESTARTS   AGE
image-automation-controller-6c4fb698d4-zrp78   1/1     Running   0          29s
image-reflector-controller-5dfa39212d-hnnvj    1/1     Running   0          29s
kustomize-controller-424f5ab2a2-u2hwb          1/1     Running   0          29s
source-controller-2wc41z892-axkr1              1/1     Running   0          29s
```

Since our helloenv-app repository is public, application will get deployed as part of the bootstrap step. You can check both staging and prod environments with following commands.
```sh
kubectl rollout status -n helloenv-staging deployments
watch kubectl get pods -n helloenv-staging

kubectl rollout status -n helloenv-prod deployments
watch kubectl get pods -n helloenv-prod
```

#### Setup Git authentication for Flux

After bootstrapping Flux, we will grant it write access to our GitHub repositories. This will allow Flux to update image tags in manifests, create pull requests etc.

The [`flux create secret git`](https://fluxcd.io/flux/cmd/flux_create_secret_git/) command creates an SSH key pair for the specified host and puts it into a named Kubernetes secret in Flux's management namespace (by default flux-system). The command also outputs the public key, which should be added to the forked repo's "Deploy keys" in GitHub.


```sh
GITHUB_USER=<your_github_username>
flux create secret git ssh-credentials \
  --url=ssh://git@github.com/${GITHUB_USER}/helloenv-app
```

If you need to retrieve the public key later, you can extract it from the secret as follows:

```sh
kubectl get secret ssh-credentials -n flux-system -ojson \
  | jq -r '.data."identity.pub"' | base64 -d
```

Use the public key as a Deploy key in your fork of the helloenv-app repo. Browse to the following URL, replacing `<your_github_username>` with your GitHub username: `https://github.com/<your_github_username>/helloenv-app/settings/keys`.   
The page will appear as follows.     

![Deploy keys page](/assets/img/Blog/automatic-image-update-to-git-using-flux-github-actions/deploy-keys-page.png)

Click "Add deploy key" and paste the key data (starts with `ssh-<alg>...` ) into the contents. The name is arbitrary, we use helloenv-app-secret here.

### Accessing the application

The default image version can be checked by accessing our application using curl command. If you are using minikube, make sure you enable the ingress addon so the ingress functionality works.  

```sh
minikube addons enable ingress
```  

If you are running on the local cluster, you will have to add the minikube IP to your `/etc/hosts` file. The command `minikube ip` will give the IP of your cluster.  

For example, the entry in `/etc/hosts` file will look like this:

```sh
192.168.49.2 helloenv.prod.com helloenv.stage.com
```

You can check the ingress that is available for stage and prod.

```
$ kubectl get ingress --all-namespaces

NAMESPACE          NAME       CLASS    HOSTS                ADDRESS   PORTS   AGE
helloenv-prod      helloenv   <none>   helloenv.prod.com              80      9m22s
helloenv-staging   helloenv   <none>   helloenv.stage.com             80      9m21s
```

Now, you can access the application with following commands.

```sh
curl helloenv.prod.com

curl helloenv.stage.com
```

The result of these `curl` commands will show the current default image versions that the deployment is using.

### Releasing to stage 

Now, let's make some changes to the app and release it to the stage. We can make a minor change to the [app.py](https://github.com/infracloudio/flux-helloenv-app/blob/main/app/app.py) file in the application code repository `helloenv-app`, by changing the message and pushing it to the main branch, of course, in real case it will be pushed to main through a PR.  

This change triggers the [CI](https://github.com/infracloudio/flux-helloenv-app/blob/main/.github/workflows/ci.yaml) job configured on the application code repository. This job contains the logic to check the tag and then create the image tag based on it.

```yaml
run: |
  if [[ ${{ github.event.ref }} =~ ^refs/tags/[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "IMAGE_ID=${{ github.ref_name }}" >> "$GITHUB_OUTPUT"
  else
    ts=$(date +%s)
    branch=${GITHUB_REF##*/}
    echo "IMAGE_ID=${branch}-${GITHUB_SHA::8}-${ts}" >> "$GITHUB_OUTPUT"
  fi
```

From this snippet, it can be seen that it checks for the condition for prod and stage, as well as how we set the tagging of images.  

So, since we are doing this for stage, it will tag the image as `${branch}-${GITHUB_SHA::8}-${ts}`.

A useful tag format is `<branch>-<sha1>-<timestamp>`.  

Including the branch information with an image makes it easier to trace the source code's branch and commit associated with that image. Additionally, having the branch information and [unix time](https://en.wikipedia.org/wiki/Unix_time) allows you to filter for images originating from a [specific branch when needed](https://fluxcd.io/flux/guides/sortable-image-tags/).  

Since this CI stage for Staging will push the image to the image repository, the image update and image policy will kick in and replace the image with the latest built image based on timestamp.  

#### Check status of stage environment

We can check the status of image automation and image policy for staging.

```sh
$ flux get image all

NAME                    	LAST SCAN           	SUSPENDED	READY	MESSAGE                       
imagerepository/helloenv	2023-10-17T12:19:26Z	False    	True 	successful scan: found 9 tags	

NAME                        	LATEST IMAGE                                       	READY	MESSAGE                                                                                
imagepolicy/helloenv-prod   	docker.io/shapai/helloenv:1.0.2                    	True 	Latest image tag for 'docker.io/shapai/helloenv' resolved to 1.0.2                    
imagepolicy/helloenv-staging	docker.io/shapai/helloenv:main-b470560d-1696668188	True 	Latest image tag for 'docker.io/shapai/helloenv' resolved to Build-b470560d-1696668188

NAME                                  	LAST RUN            	SUSPENDED	READY	MESSAGE                                                      
imageupdateautomation/helloenv-prod   	2023-10-17T12:18:41Z	False    	True 	no updates made                                             	
imageupdateautomation/helloenv-staging	2023-10-17T12:18:39Z	False    	True 	no updates made; last commit f798e64 at 2023-10-07T09:25:32Z	
```

Let's curl the stage domain and see the output:

```sh
$ curl helloenv.stage.com

This is staging environment with (Version: main-b470560d-1696668188)
```

The output will give you the latest version, which you can verify with commit id and timestamp with which the image was created.

### Releasing to production

Now, assuming we have tested the release in stage and got go-ahead, we are ready to release in production by tagging the release.

We will be releasing it in prod by tagging the release. 

```sh
git tag -a 1.0.2 -m "prod modified"
git push --tags
```

This will, in turn, trigger the CI and create an image tag with that Git tag that we pushed.
Once this is triggered and pushed, the image automation in Flux will commit and push the changes to the flux-image-update branch.   

This will then create a PR using workflow in [helloenv-app repo](https://github.com/infracloudio/flux-helloenv-app/blob/main/.github/workflows/auto-pr.yaml). This PR needs a manual review and approval since it is a prod environment. Once this gets approved, it changes the tag with the latest tag in [prod kustomization](https://github.com/infracloudio/flux-helloenv-app/blob/main/demo/kustomize/prod/kustomization.yaml).

![PR created using workflow in helloenv-app repo](/assets/img/Blog/automatic-image-update-to-git-using-flux-github-actions/pr-created-using-helloenv-app-repo.png)

curl to the prod DNS should give you the latest tag as the version. 

```sh
$ curl helloenv.prod.com

This is production environment (Version: 1.0.2)
```

## Incident Management

During an incident, you may want to halt Flux from updating images in your Git repository. You can accomplish this by suspending image automation either in-cluster or by editing the ImageUpdateAutomation manifest in Git.

### In-cluster suspension

You can suspend image automation directly in a cluster using the following command:

```sh
flux suspend image update helloenv-prod
```

Alternatively, you can suspend image automation by editing the ImageUpdateAutomation manifest in Git. Here's an example of the manifest:

```yaml
kind: ImageUpdateAutomation
metadata:
  name: helloenv-prod
  namespace: flux-system
spec:
  suspend: true
```

### Resuming automation

Once the incident is resolved, you can resume the automation using the following command:

```sh
flux resume image update helloenv-prod
```

### Pausing automation for a specific image

If you want to pause automation for a particular image only, you can suspend and resume image scanning for that specific image. For example:

```sh
flux suspend image repository helloenv
```

### Reverting image updates

Assuming you've configured Flux to update an application to its latest stable version and an incident occurs, you can instruct Flux to revert to a previous image version.  

#### Reverting via command

For instance, to revert from version 1.0.1 to 1.0.0, you can use the following command:

```sh
flux create image policy helloenv-prod --image-ref=helloenv --select-semver=1.0.0
```

#### Reverting via Git manifest

You can also make this change by editing the ImagePolicy manifest in Git. Here's an example of the manifest:

```yaml
kind: ImagePolicy
metadata:
  name: helloenv-prod
  namespace: flux-system
spec:
  policy:
    semver:
      range: 1.0.0
```

### Updating the image policy

When a new version, e.g., 1.0.2, becomes available, you can update the policy again to consider only versions greater than 1.0.1. This can be achieved using the following command:

```sh
flux create image policy helloenv-prod --image-ref=helloenv --select-semver=">1.0.1"
```

This change will prompt Flux to update the podinfo deployment manifest in Git and roll out the specified image version in-cluster.
