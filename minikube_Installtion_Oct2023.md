Ref: https://docs.docker.com/engine/install/ubuntu/

# Installing Minikube on Ubuntu Linux

Minikube is a valuable tool for setting up a local Kubernetes cluster, primarily designed for development purposes. In this guide, we'll walk you through the process of installing Minikube on Ubuntu Linux, enabling you to establish a functional Kubernetes environment in under five minutes.

## Prerequisites

Before you begin, make sure you have the following prerequisites:

- An active instance of an Ubuntu-based Linux distribution.
- A user account with sudo privileges.
- At least two CPUs.
- A minimum of 2GB of available memory.
- A minimum of 20GB of free disk space.

## Installing Docker

Before you can use Minikube, you need to have Docker CE (Community Edition) installed. Follow these steps to install Docker:

1. Add the official Docker GPG key:

    ```bash
    $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    ```

2. Add the Docker repository:

    ```bash
    $ echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```

3. Install the necessary dependencies:

    ```bash
    $ sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release -y
    ```

4. Update the package list:

    ```bash
    $ sudo apt-get update
    ```

5. Install Docker:

    ```bash
    $ sudo apt-get install docker-ce docker-ce-cli containerd.io -y
    ```

6. Add your user to the docker group:

    ```bash
    $ sudo usermod -aG docker $USER
    ```

## Installing Minikube

Now, let's install Minikube:

1. Download the latest Minikube binary:

    ```bash
    $ wget https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    ```

2. Copy the Minikube binary to `/usr/local/bin`:

    ```bash
    $ sudo cp minikube-linux-amd64 /usr/local/bin/minikube
    ```

3. Give the Minikube executable the proper permissions:

    ```bash
    $ sudo chmod +x /usr/local/bin/minikube
    ```

## Installing kubectl

Next, we need to install the `kubectl` command-line utility:

1. Download the binary executable file:

    ```bash
    $ curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
    ```

2. Give the `kubectl` binary executable permission:

    ```bash
    $ chmod +x kubectl
    ```

3. Move the `kubectl` binary to `/usr/local/bin`:

    ```bash
    $ sudo mv kubectl /usr/local/bin/
    ```

## Starting Minikube

You can now start Minikube with the following command:

```bash
$ minikube start --driver=docker
