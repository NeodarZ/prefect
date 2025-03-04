---
description: Learn how to store your flow code in a Docker image and serve your flow on any docker-compatible infrastructure.
tags:
    - Docker
    - containers
    - orchestration
    - infrastructure
    - deployments
    - images
    - Kubernetes
search:
  boost: 2
---

# Running Flows with Docker

In the [Deployments](/tutorial/deployments/) tutorial, we looked at serving a flow that enables scheduling or creating flow runs via the Prefect API.

With our Python script in hand, we can build a Docker image for our script, allowing us to serve our flow in various remote environments. We'll use Kubernetes in this guide, but you can use any Docker-compatible infrastructure.

In this guide we'll:

- Write a Dockerfile to build an image that stores our Prefect flow code.
- Build a Docker image for our flow.
- Deploy and run our Docker image on a Kubernetes cluster.
- Look at the Prefect-maintained Docker images and discuss options for use

Note that in this guide we'll create a Dockerfile from scratch. Alternatively, Prefect makes it convenient to build a Docker image as part of deployment creation. You can even include environment variables and specify additional Python packages to install at runtime.

If creating a deployment with a `prefect.yaml` file, the build step makes it easy to customize your Docker image and push it to the registry of your choice. See an example [here](/guides/deployment/kubernetes/#define-a-deployment).

Deployment creation with a Python script that includes `flow.deploy` similarly allows you to customize your Docker image with keyword arguments as shown below.

```python
...

if __name__ == "__main__":
    hello_world.deploy(
        name="my-first-deployment",
        work_pool_name="above-ground",
        image='my_registry/hello_world:demo',
        job_variables={"env": { "EXTRA_PIP_PACKAGES": "boto3" } }
    )
```

## Prerequisites

To complete this guide, you'll need the following:

- A Python script that defines and serves a flow.
  - We'll use the flow script and deployment from the [Deployments](/tutorial/deployments/) tutorial.
- Access to a running Prefect API server.
  - You can sign up for a forever free [Prefect Cloud account](https://docs.prefect.io/cloud/) or run a Prefect API server locally with `prefect server start`.
- [Docker Desktop](https://docs.docker.com/desktop/) installed on your machine.

## Writing a Dockerfile

First let's make a clean directory to work from, `prefect-docker-guide`.

```bash
mkdir prefect-docker-guide
cd prefect-docker-guide
```

In this directory, we'll create a sub-directory named `flows` and put our flow script from the [Deployments](/tutorial/deployments/) tutorial in it.

```bash
mkdir flows
cd flows
touch prefect-docker-guide-flow.py
```

Here's the flow code for reference:

```python title="prefect-docker-guide-flow.py"
import httpx
from prefect import flow


@flow(log_prints=True)
def get_repo_info(repo_name: str = "PrefectHQ/prefect"):
    url = f"https://api.github.com/repos/{repo_name}"
    response = httpx.get(url)
    response.raise_for_status()
    repo = response.json()
    print(f"{repo_name} repository statistics 🤓:")
    print(f"Stars 🌠 : {repo['stargazers_count']}")
    print(f"Forks 🍴 : {repo['forks_count']}")


if __name__ == "__main__":
    get_repo_info.serve(name="prefect-docker-guide")
```

The next file we'll add to the `prefect-docker-guide` directory is a `requirements.txt`. We'll include all dependencies required for our `prefect-docker-guide-flow.py` script in the Docker image we'll build.

```bash
touch requirements.txt
```

Here's what we'll put in our `requirements.txt` file:

```txt title="requirements.txt"
prefect>=2.12.0
httpx
```

Next, we'll create a `Dockerfile` that we'll use to create a Docker image that will also store the flow code.

```bash
touch Dockerfile
```

We'll add the following content to our `Dockerfile`:

```dockerfile title="Dockerfile"
# We're using the latest version of Prefect with Python 3.10
FROM prefecthq/prefect:2-python3.10

# Add our requirements.txt file to the image and install dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt --trusted-host pypi.python.org --no-cache-dir

# Add our flow code to the image
COPY flows /opt/prefect/flows

# Run our flow script when the container starts
CMD ["python", "flows/prefect-docker-guide-flow.py"]
```

## Building a Docker image

Now that we have a Dockerfile we can build our image by running:

```bash
docker build -t prefect-docker-guide-image .
```

We can check that our build worked by running a container from our new image.

=== "Cloud"

    Our container will need an API URL and and API key to communicate with Prefect Cloud. 
    
    - You can get an API key from the [API Keys](https://docs.prefect.io/2.12.0/cloud/users/api-keys/) section of the user settings in the Prefect UI. 

    - You can get your API URL by running `prefect config view` and copying the `PREFECT_API_URL` value.

    We'll provide both these values to our container by passing them as environment variables with the `-e` flag.

    ```bash
    docker run -e PREFECT_API_URL=YOUR_PREFECT_API_URL -e PREFECT_API_KEY=YOUR_API_KEY prefect-docker-guide-image
    ```

    After running the above command, the container should start up and serve the flow within the container!

=== "Self-hosted"

    Our container will need an API URL and network access to communicate with the Prefect API. 
    
    For this guide, we'll assume the Prefect API is running on the same machine that we'll run our container on and the Prefect API was started with `prefect server start`. If you're running a different setup, check out the [Hosting a Prefect server guide](/guides/host/) for information on how to connect to your Prefect API instance.
    
    To ensure that our flow container can communicate with the Prefect API, we'll set our `PREFECT_API_URL` to `http://host.docker.internal:4200/api`. If you're running Linux, you'll need to set your `PREFECT_API_URL` to `http://localhost:4200/api` and use the `--network="host"` option instead.

    ```bash
    docker run --network="host" -e PREFECT_API_URL=http://host.docker.internal:4200/api prefect-docker-guide-image
    ```

    After running the above command, the container should start up and serve the flow within the container!

## Deploying to a remote environment

Now that we have a Docker image with our flow code embedded, we can deploy it to a remote environment!

For this guide, we'll simulate a remote environment by using Kubernetes locally with Docker Desktop. You can use the [instructions provided by Docker to set up Kubernetes locally.](https://docs.docker.com/desktop/kubernetes/)

### Creating a Kubernetes deployment manifest

To ensure the process serving our flow is always running, we'll create a [Kubernetes deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/). If our flow's container ever crashes, Kubernetes will automatically restart it, ensuring that we won't miss any scheduled runs.

First, we'll create a `deployment-manifest.yaml` file in our `prefect-docker-guide` directory:

```bash
touch deployment-manifest.yaml
```

And we'll add the following content to our `deployment-manifest.yaml` file:

=== "Cloud"

    ```yaml title="deployment-manifest.yaml"
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: prefect-docker-guide
    spec:
      replicas: 1
      selector:
        matchLabels:
          flow: get-repo-info
      template:
        metadata:
          labels:
            flow: get-repo-info
        spec:
          containers:
          - name: flow-container
            image: prefect-docker-guide-image:latest
            env:
            - name: PREFECT_API_URL
              value: YOUR_PREFECT_API_URL
            - name: PREFECT_API_KEY
              value: YOUR_API_KEY
            # Never pull the image because we're using a local image
            imagePullPolicy: Never
    ```

    !!!tip "Keep your API key secret"
          In the above manifest we are passing in the Prefect API URL and API key as environment variables. This approach is simple, but it is not secure. If you are deploying your flow to a remote cluster, you should use a [Kubernetes secret](https://kubernetes.io/docs/concepts/configuration/secret/) to store your API key.

=== "Self-hosted"

    ```yaml title="deployment-manifest.yaml"
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: prefect-docker-guide
    spec:
      replicas: 1
      selector:
        matchLabels:
          flow: get-repo-info
      template:
        metadata:
          labels:
            flow: get-repo-info
        spec:
          containers:
          - name: flow-container
            image: prefect-docker-guide-image:latest
            env:
            - name: PREFECT_API_URL
              value: <http://host.docker.internal:4200/api>
            # Never pull the image because we're using a local image
            imagePullPolicy: Never
    ```

    !!!tip "Linux users"
        If you're running Linux, you'll need to set your `PREFECT_API_URL` to use the IP address of your machine instead of `host.docker.internal`.

This manifest defines how our image will run when deployed in our Kubernetes cluster. Note that we will be running a single replica of our flow container. If you want to run multiple replicas of your flow container to keep up with an active schedule, or because our flow is resource-intensive, you can increase the `replicas` value.

### Deploying our flow to the cluster

Now that we have a deployment manifest, we can deploy our flow to the cluster by running:

```bash
kubectl apply -f deployment-manifest.yaml
```

We can monitor the status of our Kubernetes deployment by running:

```bash
kubectl get deployments
```

Once the deployment has successfully started, we can check the logs of our flow container by running the following:

```bash
kubectl logs -l flow=get-repo-info
```

Now that we're serving our flow in our cluster, we can trigger a flow run by running:

```bash
prefect deployment run get-repo-info/prefect-docker-guide
```

If we navigate to the URL provided by the `prefect deployment run` command, we can follow the flow run via the logs in the Prefect UI!

## Prefect-maintained Docker images

Every release of Prefect results in several new Docker images.
These images are all named [prefecthq/prefect](https://hub.docker.com/r/prefecthq/prefect) and their
**tags** identify their differences.

### Image tags

When a release is published, images are built for all of Prefect's supported Python versions.
These images are tagged to identify the combination of Prefect and Python versions contained.
Additionally, we have "convenience" tags which are updated with each release to facilitate automatic updates.

For example, when release `2.11.5` is published:

1. Images with the release packaged are built for each supported Python version (3.8, 3.9, 3.10, 3.11) with both standard Python and Conda.
2. These images are tagged with the full description, e.g. `prefect:2.1.1-python3.10` and `prefect:2.1.1-python3.10-conda`.
3. For users that want more specific pins, these images are also tagged with the SHA of the git commit of the release, e.g. `sha-88a7ff17a3435ec33c95c0323b8f05d7b9f3f6d2-python3.10`
4. For users that want to be on the latest `2.1.x` release, receiving patch updates, we update a tag without the patch version to this release, e.g. `prefect.2.1-python3.10`.
5. For users that want to be on the latest `2.x.y` release, receiving minor version updates, we update a tag without the minor or patch version to this release, e.g. `prefect.2-python3.10`
6. Finally, for users who want the latest `2.x.y` release without specifying a Python version, we update `2-latest` to the image for our highest supported Python version, which in this case would be equivalent to `prefect:2.1.1-python3.10`.

!!! tip "Choose image versions carefully"
    It's a good practice to use Docker images with specific Prefect versions in production.

    Use care when employing images that automatically update to new versions (such as `prefecthq/prefect:2-python3.11` or `prefecthq/prefect:2-latest`).

### Standard Python

Standard Python images are based on the official Python `slim` images, e.g. `python:3.10-slim`.

| Tag                   |       Prefect Version       | Python Version  |
| --------------------- | :-------------------------: | -------------:  |
| 2-latest              | most recent v2 PyPi version |            3.10 |
| 2-python3.11          | most recent v2 PyPi version |            3.11 |
| 2-python3.10          | most recent v2 PyPi version |            3.10 |
| 2-python3.9           | most recent v2 PyPi version |            3.9  |
| 2-python3.8           | most recent v2 PyPi version |            3.8  |
| 2.X-python3.11        |             2.X             |            3.11 |
| 2.X-python3.10        |             2.X             |            3.10 |
| 2.X-python3.9         |             2.X             |            3.9  |
| 2.X-python3.8         |             2.X             |            3.8  |
| sha-&lt;hash&gt;-python3.11 |            &lt;hash&gt;           |            3.11 |
| sha-&lt;hash&gt;-python3.10 |            &lt;hash&gt;           |            3.10 |
| sha-&lt;hash&gt;-python3.9  |            &lt;hash&gt;           |            3.9  |
| sha-&lt;hash&gt;-python3.8  |            &lt;hash&gt;           |            3.8  |

### Conda-flavored Python

Conda flavored images are based on `continuumio/miniconda3`.
Prefect is installed into a conda environment named `prefect`.

| Tag                         |       Prefect Version       | Python Version  |
| --------------------------- | :-------------------------: | -------------:  |
| 2-latest-conda              | most recent v2 PyPi version |            3.10 |
| 2-python3.11-conda          | most recent v2 PyPi version |            3.11 |
| 2-python3.10-conda          | most recent v2 PyPi version |            3.10 |
| 2-python3.9-conda           | most recent v2 PyPi version |            3.9  |
| 2-python3.8-conda           | most recent v2 PyPi version |            3.8  |
| 2.X-python3.11-conda        |             2.X             |            3.11 |
| 2.X-python3.10-conda        |             2.X             |            3.10 |
| 2.X-python3.9-conda         |             2.X             |            3.9  |
| 2.X-python3.8-conda         |             2.X             |            3.8  |
| sha-&lt;hash&gt;-python3.11-conda |            &lt;hash&gt;           |            3.11 |
| sha-&lt;hash&gt;-python3.10-conda |            &lt;hash&gt;           |            3.10 |
| sha-&lt;hash&gt;-python3.9-conda  |            &lt;hash&gt;           |            3.9  |
| sha-&lt;hash&gt;-python3.8-conda  |            &lt;hash&gt;           |            3.8  |

## Building your own image

If your flow relies on dependencies not found in the default `prefecthq/prefect` images, you may want to build your own image. You can either
base it off of one of the provided `prefecthq/prefect` images, or build your own image.
See the [Work pool deployment guide](/guides/prefect-deploy/) for discussion of how Prefect can help you build custom images with dependencies specifiied in a `requirements.txt` file.

By default, Prefect [work pools](/concepts/work-pools) that use containers refer to the `2-latest` image.
You can specify another image at work pool creation.
The work pool image choice can be overridden in individual deployments.

### Extending the `prefecthq/prefect` image manually

Here we provide an example `Dockerfile` for building an image based on
`prefecthq/prefect:2-latest`, but with `scikit-learn` installed.

```dockerfile
FROM prefecthq/prefect:2-latest

RUN pip install scikit-learn
```

### Choosing an image strategy

The options described above have different complexity (and performance) characteristics. For choosing a strategy, we provide the following recommendations:

- If your flow only makes use of tasks defined in the same file as the flow, or tasks that are part of `prefect` itself, then you can rely on the default provided `prefecthq/prefect` image.

- If your flow requires a few extra dependencies found on PyPI, you can use the default `prefecthq/prefect` image and set `prefect.deployments.steps.pip_install_requirements:` in the `pull`step to install these dependencies at runtime.

- If the installation process requires compiling code or other expensive operations, you may be better off building a custom image instead.

- If your flow (or flows) require extra dependencies or shared libraries, we recommend building a shared custom image with all the extra dependencies and shared task definitions you need. Your flows can then all rely on the same image, but have their source stored externally. This option can ease development, as the shared image only needs to be rebuilt when dependencies change, not when the flow source changes.

## Next steps

We only served a single flow in this guide, but you can extend this setup to serve multiple flows in a single Docker image by updating your Python script to using `flow.to_deployment` and `serve` to [serve multiple flows or the same flow with different configuration](/concepts/flows#serving-multiple-flows-at-once).

To learn more about deploying flows, check out the [Deployments](/concepts/deployments/) concept doc!

For advanced infrastructure requirements, such as executing each flow run within its own dedicated Docker container, learn more in the [Work pool deployment guide](/guides/prefect-deploy/).
