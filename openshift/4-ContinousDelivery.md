# Continuous Delivery with Argo CD

Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes. The deployment environment is a namespace in a container platform like Kubernetes or Red Hat OpenShift.

Argo CD models a collection of applications as a project and uses a Git repository to store the application's desired state.

Argo CD is flexible in the structure of the application configuration represented in the Git repository.

Argo CD supports defining Kubernetes manifests in a number of ways:

* helm charts
* kustomize
* ksonnet
* jsonnet
* plain directory of yaml/json manifests
* custom plugins

Argo CD compares the actual state of the application in the cluster with the desired state defined in Git and determines if they are out of sync. When it detects the environment is out of sync, Argo CD can be configured to either send out a notification to kick off a separate reconciliation process or Argo CD can automatically synchronize the environments to ensure they match.

---
**Note**

Confidential information like passwords and security tokens should not be checked into the Git repository. Managing secrets in Argo CD (!!!provide link!!!) provides information on how to handle confidential information in the GitOps repo.

---

## Configuring GitOps with Argo CD

**Terminology:**

Argo CD uses a number of terms to refer to the components.

* Application - A deployable unit

    In the context of the environment, an application is one Helm chart that contains one container image that was produced by one CI pipeline. While Helm charts and images could certainly be combined to make more sophisticated applications in more advanced scenarios, we will be using this simple definition here.

* Project - A collection of applications that make up a solution

## Set up the GitOps repo

Argo CD uses a Git repo to express the desired state of the Kubernetes environment. The basic setup uses one repository to represent one project. Within that repository, each application that makes up the project will be described in its own folder. The repository will also contain a branch for each destination (i.e. cluster and namespace) into which we want to deploy the applications.

---
**Note**

There is nothing special about a git repository used for git-ops. All that is required at a minimum is a hosted git repository that is accessible from by the Argo CD instance. The Argo CD Starter Kit used in the following steps is optional and provides some application templates to help simplify some configuration activities.

--

1. Create a new repo from the [Argo CD Starter Kit](https://github.com/IBM/template-argocd-gitops/generate). If you see a 404 error when you click on the link, you need to sign in to github.

2. Clone the project to your machine

```bash
git clone ${GIT_URL_GITOPS}
```

3. navigate into the directory

```bash
cd ${GIT_DIRECTORY}
```

4. Create and push test branch

```bash
git checkout -b test
git push -u origin test
```

## Hook the CI pipeline to the CD pipeline

The last stage in the CI pipeline updates a GitOps repository with the updated application metadata from the build. In order to do that, the CI pipeline needs to know which repository should be used and needs the credentials to push changes to that repository. As with other configuration within the pipeline, this is handled with config maps and secrets:

* A secret named `git-credentials` holds the credentials the CI pipeline uses to access all the respositories in the Git host (e.g. GitHub, GitLab, BitBucket, etc. If you used the IGC CLI to register the pipeline then this secret has already been created.
* A config map named `gitops-repo` holds the url and branch for the gitops repository.

Fortunately the IGC CLI provides a gitops command to simplify this step. Information on how to use the command as well as the alternative manual steps can be found in the IGC CLI gitops command section.

1. Make sure to switch context to the project/namespace CI namespace

```bash
oc project ${DEV_NAMESPACE}
```

2. Run the gitops command to create the config map and secret in the CI namespace

```bash
igc gitops
```

---
**Note**

* For the secret to be available to the CI pipeline, the secret needs to be created in the same namespace where the pipeline is running.
* The value provided for branch is the one the pipeline will use to when committing changes to trigger the CD pipeline.

---

As of v2.0.0 of the Tekton tasks and the Jenkins pipelines, the CI pipeline will create a folder and the initial configuration for an application deployment if it doesn't already exist. This means, there is no other manual configuration required to set up the repository.

Now run a new Pipeline and make sure a directory for the application is created on the gitops git repository. This is required before configuring ArgoCD.

## Configure Release namespaces

ArgoCD will deploy the application into the "releases" namespace such as ${TEST_NAMESPACE} or ${STAGING_NAMESPACE}.

1. Creat a release namespace where ArgoCD will deploy the application

```bash
oc new-project ${TEST_NAMESPACE}
```

where `${TEST_NAMESPACE}` is the name you've chosen for your testing namespace. The release namespaces need pull secrets for the application container images to be pulled. In this workshop we are using the IBM Container Image Registry. For this, copy the pull secret `all-icr-io` from the `default` namespace and then add this secret to the service account used by your application (ie `default` service account)

2. Use the Toolkit CLI to copy the secret and setup the service account

```bash
igc pull-secret ${TEST_NAMESPACE} -t default -z default
```

## Register the GitOps repo in ArgoCD

Now that the repository has been created, we need to tell ArgoCD where it is. For this, open the Developer Dashboard and click the ArgoCD link to open ArgoCD. 

1. Log into ArgoCD

2. Click on the gear icon on the left menu to access the Settings options

    ![ArgoCD config](images/argocd-config-repo.png)

3. Select the `Repositories` option

4. Click either the `Connect Repo using HTTPS` or `Connect Repo using SSH` button at the top and provide the information for the GitOps repo you just created. For `HTTPS` you can use the access token you used when you ran `igc gitops`.

## Create a project in Argo CD

In Argo CD terms, each deployable component is an application and applications are grouped into projects. Projects are not required for Argo CD to be able to deploy applications, but it helps to organize applications and provide some restrictions on what can be done for applications that make up a project.


1. Log into the Argo CD user interface

2. Click on the gear icon on the left menu to access the Settings options

    ![Argo CD config project]()images/argocd-config-project.png

3. Select the Projects option

4. Press the New Project button at the top of the page

5. Specify the properties for the new project

    * Name - Provide the name for the project
        * Description - A brief description of the project
        * Source - Press Add source and pick the Git repository from the list that was added previously
        * Destinations - Add `https://kubernetes.default.svc` for the cluster url and ${TEST_NAMESPACE} for the namespace
        * Press Create

## Add an application in Argo CD for each application component

---
**Warning**

Before continuing to setup ArgoCD, please verify that the CI Pipeline run created the directory for the application on the gitops git repository and the directory container the helm related files including requirements.yaml

---

The last step in the process is to define the application(s) within Argo CD that should be managed. This consists of connecting the config within the Git repo to the cluster and namespace.

1. Log into Argo CD user interface

2. Press `New Application` and provide the following values:
    * `application name` - The name of the application. It is recommended to use the format of `{namespace}-{image name}`
        * `project` - The ArgoCD project with which the application should be included
        * `sync-policy` - The manner with which Argo CD will use to manage the deployed artifacts. `Automatic` is recommended
        * `repository url` - The Git url where the configuration is stored (restricted to git urls configured in the Argo Project)
        * `revision` - The Git branch where the configuration for this instance is stored
        * `path` - The path within the repository where the application config is located (should be the application name)
        * `destination cluster` - The cluster url for the deployment
        * `destination namespace` - The namespace where the application should be deployed (restricted to namespaces configured in the Argo Project)

3. Repeat that step for each application and each environment