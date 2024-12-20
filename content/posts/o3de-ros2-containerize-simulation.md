+++
title = "Running robotic simulations using Docker Images"
draft = true
date = "2024-12-16"
description = ""
tags = [
    "AWS",
    "Simulation",
    "Robotics",
    "Docker",
    "ROS"
]
+++


# Distributing Simulation Projects
For robotics simulation projects for ROS2 and O3DE, collaboration on constructing and refining the simulation can be shared across a team through source control such as GIT or Perforce. However, in order iterate through developing the simulation, O3DE, ROS2, and its environment and dependencies need to be installed separately in order to access and contribute to the project. I explained how Amazon's EC2 helps with the environment through a marketplace AMI in my previous blog post. 

The primary goal of simulations is for training robots through ROS2, not through O3DE. Once the simulation is ready for use, O3DE itself is not needed to run the simulation. The simulation project can be exported into a launcher application separate from O3DE's Editor and build environment. Exporting a simulation project into its own launcher will package just the launcher executable and its bare minimum dependent binaries, assets, and necessary settings needed to launch the simulation. All the additional Editor and Asset Processor related artifacts that comes with a full O3DE installation will be omitted.

### Containerization

The exported project alone usually is not enough to have it run on different machines. Along with the base operating system (in this case, Ubuntu Linux) there are additional pre-requisite package requirements. In order to have the simulation project run consistently across different machines, each machine must have the same operating system, packages, and environment. Another approach to utilize containerization. 

Containerization provides many benefits for our use case:

#### Portability
The simulation project will be contained in its own isolated environment and operating system. The specific operating system, Ubuntu 22.04, and all of its required packages will be embedded in the container image and isolated from the host machine. That means no additional simulation-specific pre-requisites are needed to launch the simulation. This allows it to run on multiple devices and operating systems.

#### Consistency
Container images are immutable. This ensures that the application will run in a consistent manner across any environment or operating system.

#### Scalability
Containers are lightweight and run efficiently. Unlike virtual machines, they do not need to be boot up to load the operating system. Containers can be distributed and managed easily through different container orchestration tools as well. 

We will be using the same simulation project we created from my previous blog in this example.

## Exporting the Project
Once the simulation project is working as expected through the Editor and from command line, we can use the O3DE to export the project to a single folder. Through the Project Manager, click on the context button for the project **MyRobotSimulation** and click on **Open Export Settings...**

![O3DE Project Manager](/images/blog/o3de-ros2-export-project/project-manager-context.png)

This will bring up the export settings specific to this project. This is where we will configure the settings we need to export just the simulation (game) launcher.

![Export Setting](/images/blog/o3de-ros2-export-project/export_settings.png)

The export settings is designed for any type of project, not just simulations. There will only be a few settings that we need to enable and configure for our simulation launcher:

|  |  | |
| ----- | ----- | ----- |
| **Project Build Configuration**  | | profile |
| **Build Assets** | | true |
| **Level Names** | | DemoLevel |
| **Max Size** | | 2048 |
| **Asset Bundling Path** | | build/asset_bundling |
| **Archive Format** | | none |
| **Build Game Launcher** | | true |
| **Build Server Launcher** | | false |
| **Build Unified Launcher** | | false |
| **Build Headless Server Launcher** | | false |

Once the configuration is set, click on **Save** to save the settings. To run the project export, click on the context menu button for the project again, and click on **Export Launcher**->**Linux**

![Export Linux](/images/blog/o3de-ros2-export-project/export-launcher-linux.png)

When the process is complete, the exported project will be located in folder named **MyRobotSimulationGamePackage** in the path **~/O3DE/Projects/MyRobotSimulation/build/export/**

# Creating a Docker Image
The exported project can now be stored into a compressed archive and distributed to other Ubuntu 22.04 hosts to use. However, in order to run the project successfully, they must conform to the same general requirements for O3DE and ROS2. This means that the runtime environment must closely match the same environment as O3DE (minus the build tools). Another approach to distribution is to use containerization. Containers gives us a portable and consistent mechanism to distribute and execute our simulation across different machines and environments. 

For this blog, we will use [Docker](https://www.docker.com/) and the [NVIDIA Container toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker). 

### Installing Docker Engine

Install Docker Engine for Ubuntu using the [Docker Installation instructions](https://docs.docker.com/engine/install/ubuntu/). The following is a quick cheat-sheet to install Docker Engine using the **apt** repository based on the [Dockers's apt instructions](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository).

#### 1. Set up Docker's **apt** repository
```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
#### 2. Install the Docker packages
```bash
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### 3. Create a **docker** group (if not already created) Add the current user to the **docker** group
```bash
sudo groupadd docker

sudo usermod -aG docker $USER

newgrp docker
```

#### 4. Verify that you can run docker without sudo by running the hello-world image.

```bash
docker run hello-world
```

> **Note:** If you run into any issues during these steps, refer to the [Docker Installation instructions](https://docs.docker.com/engine/install/ubuntu/) for help.

### Installing NVIDIA Container Toolkit
In addition to Docker Engine, you will need the NVIDIA Container toolkit in order to run the simulation within the Docker Image. The following is another quick cheat-sheet to installing the toolkit based on [NVIDIA's apt instructions](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#installing-with-apt).

#### 1. Set up NVIDIA's production repository
```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```
#### 2. Update the package list from the repository
```bash
sudo apt-get update
```
#### 3. Install the toolkit from apt
```bash
sudo apt-get install -y nvidia-container-toolkit
```

#### 4. Configure the toolkit for Docker (rootless)
```bash
nvidia-ctk runtime configure --runtime=docker --config=$HOME/.config/docker/daemon.json

systemctl --user restart docker

sudo nvidia-ctk config --set nvidia-container-cli.no-cgroups --in-place
```

### Create the Docker Image

A **Dockerfile** is the recipe for the docker engine to build a Docker Image. The Docker Image will include the minimal required packages for O3DE launchers and ROS2, based on the type of simulation that **MyRobotSimulation** was built for. Docker uses a path to refer to as its context root, and in our example, we will use the path to the exported project from earlier (**~/O3DE/Projects/MyRobotSimulation/build/export/**). 


#### 1. Create a Dockerfile

Navigate to the path **~/O3DE/Projects/MyRobotSimulation/build/export/** and create a file named **Dockerfile** with the following contents:

```docker

# This image will be based on ROS's ros humble base image
FROM ros:humble-ros-base-jammy

ENV WORKSPACE=/data/workspace

WORKDIR $WORKSPACE

RUN apt-get update && apt-get upgrade -y

# Get the necessary packages needed for the simulation launcher
RUN apt-get install -y --no-install-recommends libglu1-mesa \
                       libfontconfig1 \
                       libxcb-image0 \
                       libxcb-keysyms1 \
                       libxcb-render-util0 \
                       libxcb-randr0 \
                       libxcb-xinerama0 \
                       libxcb-xinput0 \ 
                       libxcb-xkb1 \
                       libxcb-xfixes0 \
                       libxkbcommon-x11-0 \
                       libunwind8 \ 
                       libxcb-icccm4 \
                       libnvidia-gl-470 \
                       ros-${ROS_DISTRO}-slam-toolbox \
                       ros-${ROS_DISTRO}-navigation2 \
                       ros-${ROS_DISTRO}-nav2-bringup \
                       ros-${ROS_DISTRO}-pointcloud-to-laserscan \
                       ros-${ROS_DISTRO}-gazebo-msgs \
                       ros-${ROS_DISTRO}-ackermann-msgs \
                       ros-${ROS_DISTRO}-rmw-cyclonedds-cpp \
                       ros-${ROS_DISTRO}-control-toolbox \
                       ros-${ROS_DISTRO}-nav-msgs \
    && rm -rf /var/lib/apt/lists/* 

# Copy over the exported project into this image
COPY MyRobotSimulationGamePackage $WORKSPACE/Simulation

# Enable the necessary environments for the NVIDIA toolkit and ROS2
ENV NVIDIA_VISIBLE_DEVICES=all
ENV NVIDIA_DRIVER_CAPABILITIES=all
ENV RMW_IMPLEMENTATION=rmw_cyclonedds_cpp 

```

#### 2. Build the docker image
Once the Dockerfile recipe is saved, you can use the Docker Engine to build an OCI-compliant container image. We will name the image **my_robot_simulation** (Docker is case sensitive and does not allow for mixed casing in the name) and use the **latest** tag for our sample.

Use the following command to build the image.

```bash
docker build -t my_robot_simulation:latest ~/O3DE/Projects/MyRobotSimulation/build/export
```

#### 3. Test the docker image locally
Before uploading and distributing the Docker Image, we will test it locally by running in through Docker Engine and using the NVIDIA Container Toolkit. 

In order for the image to have access to the local display, you need to add it to xhost.
```bash
xhost +local:root
```

To launch the simulation through Docker, run the following command


```bash
docker run --rm --network="bridge" --gpus all -e DISPLAY=:1 -v /tmp/.X11-unix:/tmp/.X11-unix -it my_robot_simulation:latest /data/workspace/Simulation/MyRobotSimulation.GameLauncher
```

While the simulation is running from the Docker Image, you can now connect to it through ROS2 using the sample navigation example from the MyRobotSimulation project (as described in the previous blog). Open up a new terminal and enter the following lines.

```bash
cd ~/O3DE/Projects/MyRobotSimulation/Examples

ros2 launch slam_navigation/launch/navigation.launch.py
```


## Publish the Docker Image
Docker images do not upload to a repository the same way as regular files or archives do. Instead, they rely on OCI-Compliant container registries. For this example, we will use [Amazon Elastic Container Registry](https://aws.amazon.com/ecr/) to upload and distribute our Docker image. 

>**Note**: As with the AWS EC2 instance from the previously blog, we will be using the same Amazon account login for ECR

Before starting, make sure that you have the [AWS CLI](https://aws.amazon.com/cli/) installed
```
sudo apt  install awscli
```

Make sure that you have your AWS credentials and region setup running `aws configure` 

#### 1. Create a repository called **my_robot_simulation**
Run the following command from the terminal to create a repository called 'my_robot_simulation'
```bash
aws ecr create-repository --repository-name my_robot_simulation
```

>**Note**: Docker enforces a strict naming convention, and does not allow for mixed casing in the name. 

You should get a response similar to the following:

```json
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:us-west-2:000000000000:repository/my_robot_simulation",
        "registryId": "000000000000",
        "repositoryName": "my_robot_simulation",
        "repositoryUri": "000000000000.dkr.ecr.us-west-2.amazonaws.com/my_robot_simulation",
        "createdAt": 1734476138.426,
        "imageTagMutability": "MUTABLE",
        "imageScanningConfiguration": {
            "scanOnPush": false
        },
        "encryptionConfiguration": {
            "encryptionType": "AES256"
        }
    }
}
```
Note the `repositoryUri` value and set it to an environment variable for convenience
```bash
export ECR_SIMULATION_URI=000000000000.dkr.ecr.us-west-2.amazonaws.com/my_robot_simulation
```

#### 2. Log into Amazon ECR with the following command

```bash
aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_SIMULATION_URI
```
You should see a message saying `Login Succeeded`

#### 3. Tag the local Docker Image for the simulation to the repository URI

```bash
docker tag my_robot_simulation $ECR_SIMULATION_URI
```

#### 4. Push the local Docker Image to the Amazon ECR for your account.

```bash
docker push $ECR_SIMULATION_URI
```
This will begin the process to push the Docker Image to ECR. Once the process is done, you can view and verify that the image has been uploaded to your account. Log into your AWS console and go to the Amazon ECR dashboard, and click on **Repositories**. You should see the new **my_robot_simulation** repository under **Private repositories**.  

![Export Linux](/images/blog/o3de-ros2-export-project/aws-console-amazon-ecr-repos.png)


## Running Docker Image on other machines
With the Docker Image stored in your Amazon ECR account, you can distribute and run the image on any Linux host provided that it has Docker or any OCI compliant runtime package as well as the AWS CLI installed. For this example, we will continue to use Docker Engine but from a different Ubuntu Linux host.

You will need to authenticate to the AWS account where the Image has been uploaded and registered to. The user or IAM must have the right permission to pull and run the image. Also, the repository URI from the publish steps above is needed to identify the Image to download. You can also get the URI from the AWS console. 

#### 1. Set the Repository URI and log into AWS

For convenience, set an environment variable to the URI. 
```bash
export ECR_SIMULATION_URI=857350499302.dkr.ecr.us-west-2.amazonaws.com/my_robot_simulation
```

Once logged into your AWS account using the same `aws configure` steps above, you can pull the Docker Image based on the repository URI.
Then get the login password for your ECR and send it to the docker login 

```bash
aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_SIMULATION_URI
```

#### 2. Pull the docker image
Pull the Docker Image from ECR to the local Docker Engine. This will make the Image ready to run. 

```bash
docker pull $ECR_SIMULATION_URI:latest
```

> **Note**: If this step is omitted, the first attempt to run the image will perform the pull operation anyways.

#### 3. Launch the Docker Image
Once the Docker Image is pulled to your local Docker Engine, launch the image the same way as described earlier when testing the image.

```bash
xhost +local:root

docker run --rm --network="bridge" --gpus all -e DISPLAY=:1 -v /tmp/.X11-unix:/tmp/.X11-unix -it $ECR_SIMULATION_URI:latest /data/workspace/Simulation/MyRobotSimulation.GameLauncher

```

This will launch the simulation from the docker image using the environment defined in the Dockerfile recipe. Since we are running on a different machine that does not have the original MyRobotSimulation project, you will need to do one of two things:

1. Copy the Examples/template folder to this machine
2. Download it from the project template at https://github.com/o3de/o3de-extras/tree/d2888c1be6f339d323316f46633d46e35816864d/Templates/Ros2ProjectTemplate/Template/Examples and follow the instructions from the [previous blog](https://spham-amzn.github.io/posts/o3de-ros2-aws-mp-ami/) (Step 6: Setup and launch RViz).

# Tearing Down
Once we are done with distributing the simulation, we can remove it from Amazon ECR. The cost that Amazon ECR incurs is based on the storage that is used by the Docker Image ( $0.10 USD per GB/Month).  The image constructed with this blog takes up about 1.7 GB of storage, which translates to about $0.17 USD per month. To delete the ECR image, log back into your AWS console and navigate to the Amazon ECR dash board. Locate the `my_robot_simulation` under **Private registry**->**Repositories**. Select it and click the **Delete** button.

# Conclusion
In this blog we introduced a portable and consistent approach to distributing an O3DE simulation project to different machines. We took a sample robotic simulation project we created in a [previous blog](https://spham-amzn.github.io/posts/o3de-ros2-aws-mp-ami/) and exported the simulation as a separate package. We used this exported package to build an OCI compliant container image using Docker Engine and published it to a container registry using Amazon ECR. Finally we demonstrated how to pull and run the simulation docker image using Docker Engine and NVIDIA container toolkit.

