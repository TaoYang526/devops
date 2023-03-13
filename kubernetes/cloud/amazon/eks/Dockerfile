FROM amazon/aws-cli:2.7.6 as aws-eks-dev

ENTRYPOINT /bin/sh
WORKDIR /work

# install some useful tools
RUN yum install -y jq gzip nano tar git unzip wget python3 make
RUN pip3 install yq
# install kubectl
RUN curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
RUN chmod +x ./kubectl && mv ./kubectl /usr/local/bin/kubectl
# install eksctl
RUN wget https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz -O - | tar xz -C /tmp
RUN mv /tmp/eksctl /usr/local/bin
# install helm
RUN wget https://get.helm.sh/helm-v3.10.2-linux-amd64.tar.gz -O - | tar -xz -C /tmp
RUN mv /tmp/linux-amd64/helm /usr/local/bin