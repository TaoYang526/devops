# Devops Tools for Amazon EKS

## Devops Environments
For better isolation between multiple devops environments, you can build a new image spawned from the image of AWS CLI on [Docker-Hub](https://hub.docker.com/r/amazon/aws-cli),
with some required binaries (aws, eksctl, kubectl etc.) and handy tools (jq, yq, gzip, tar, unzip, wget, python3 etc.).
After that, you can launch a unique container for a specified devops environment.

For more details about required binaries, please refer to [Installing eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html),
[Installing kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html).

```shell
# build image: aws-eks-dev
docker build --target aws-eks-dev kubernetes/cloud/amazon/eks -t aws-eks-dev

# start container for devops environment
docker run --name ${DEVOPS_ENV_NAME} -it -v ${PWD}:/work aws-eks-dev:latest
```
