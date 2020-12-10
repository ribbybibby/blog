---
title: "HOW TO: Create a consistent runtime for Terraform with Docker and make"
date: 2019-05-02T18:18:51+01:00
draft: false
---
Inevitably, when you start using a shared Terraform code base between more than two or three people you have to start introducing measures to ensure consistency of approach across your team.

For instance, how do you make sure everyone is running the same version of Terraform? Or that they’re initializing the backend properly? And if you’re using `null_resource` resources or `provisioner` blocks to execute scripts out-of-band, you have to ensure everyone has the right dependencies installed.

This is one approach I’ve used to address these problems, using a Docker image and a Makefile to provide a consistent environment for running Terraform locally and a wrapper around Terraform to make sure the whole team are executing it in the same way.

### Requirements
* Relatively up to date versions of Docker and docker-compose

### Repository structure
For the purposes of this post, let’s assume your Terraform code is all versioned in one repository with a structure similar to this:
```
.
├── README.md
└── terraform
    ├── project1
    │   ├── main.tf
    │   ├── outputs.tf
    │   ├── variables.tf
    │   └── vars
    │       ├── dev.tfvars
    │       ├── prod.tfvars
    │       └── staging.tfvars
    └── project2
        ├── main.tf
        ├── outputs.tf
        ├── variables.tf
        └── vars
            ├── dev.tfvars
            ├── prod.tfvars
            └── staging.tfvars
```

There’s a folder for each project in your organization under `./terraform/` containing `.tf` files and a `.tfvars` file for each environment. This is my preferred way to partition my Terraform code, but if yours is different it doesn't make much difference in the context of this guide.

### Dockerfile
The first thing I'm going to add is a `Dockerfile` to the root of the repository:
```Dockerfile
FROM alpine:3.8

ENV TERRAFORM_VERSION=0.11.13
ENV AWSCLI_VERSION=1.14.10

RUN apk add --update bash make python python-dev py-pip build-base unzip ca-certificates jq

RUN pip install awscli==$AWSCLI_VERSION --upgrade

RUN wget https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip \
    && unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip -d /usr/local/bin/

WORKDIR /terraform

ENTRYPOINT ["/bin/bash"]
```
Pretty simple. Essentially all I’m doing here is downloading my desired version of the Terraform binary to /usr/local/bin inside a simple alpine:3.8 image.

You’ll also notice that that I install `awscli` and `jq`. This is because in this particular code base I intend to run a few scripts out-of-band that require them. That's why we're using Docker: so we can be certain that when we run Terraform, we have everything we need, no matter who's running it or where.

### docker-compose
You might have noticed that I didn’t copy the contents of my `./terraform` directory into the Docker image. That's because I'm going to mount it instead so that I can make changes to the files on my local filesystem but then apply them from within the running Docker container. 

If you were including this repo inside a CI/CD pipeline, you might choose to copy the dir in to provide you with a versioned artifact. On the other hand, in that situation I think there's still something to be said for versioning your terraform code and what is essentially your terraform binary separately. More on that later.

I create `docker-compose.yml` next to my `Dockerfile`:
```yaml
version: '3'
services:
  terraform:
    build: .
    tty: true
    stdin_open: true
    environment:
      - AWS_DEFAULT_REGION=eu-west-2
    volumes:
      - ./terraform:/terraform
      - ~/.aws:/root/.aws
```

Under volumes you can see I bind the `./terraform` directory to `/terraform` on the container. I also mount the `~/.aws` directory and set the `AWS_DEFAULT_REGION` environment variable so that the AWS credentials of the user will be available and the resources in the right region will be targeted (this is all assuming your Terraform code uses AWS resources, of course).

The `tty` and `stdin_open` bools set to `true` ensure than when I eventually run the container, I get an interactive bash shell and I can cd to one of my project directories, see all the code and run Terraform if I wish.
```
$ docker-compose run terraform
bash-4.4# cd project1
bash-4.4# ls 
main.tf  outputs.tf  variables.tf  vars
bash-4.4# terraform get
bash-4.4# terraform init

Initializing the backend..
```

## Makefile
This `Makefile` wraps around the plan and apply commands, making sure they're ran in a consistent fashion.
```Makefile
ENVIRONMENT := dev
PROJECT := $(notdir $(shell pwd))
TF_BUCKET := org-tfstate-${ENVIRONMENT}

.PHONY: update init plan apply

all: update 

update:
	@terraform get -update .

init: update
	@terraform init \
        -backend=true \
        -backend-config="bucket=$(TF_BUCKET)" \
        -backend-config="key=$(PROJECT).tfstate" .

plan: init
	@-rm -f ./plan.tfout
	@terraform plan \
        -refresh=true \
        -out=./plan.tfout \
        -var-file=vars/$(ENVIRONMENT).tfvars $(ARGS) .

apply: init
	@terraform apply ./plan.tfout
	@-rm -f ./plan.tfout
```

By default the `Makefile` targets the dev environment. This can be overriden by supplying the `ENVIRONMENT` variable when `make` is ran:
```
$ docker-compose run terraform
bash-4.4# cd project1
bash-4.4# make plan ENVIRONMENT=prod
```

Additionally, if you want to append extra arguments to `plan` you can do so with `ARGS`:
```
bash-4.4# make plan ARGS="-target=aws_s3_bucket.example"
```

You can place the `Makefile` inside each project directory, alongside the `.tf` files. Or, what I prefer to do is to put it at the root of the repository as `common.makefile` and refer back to it with an `include` statement in a `Makefile` in each project. This avoids unnecessary duplication:
```
include ../../common.makefile
```

The structure now looks like this:
```
.
├── common.makefile
├── docker-compose.yml
├── Dockerfile
├── README.md
└── terraform
    ├── project1
    │   ├── main.tf
    │   ├── Makefile
    │   ├── outputs.tf
    │   ├── variables.tf
    │   └── vars
    │       ├── dev.tfvars
    │       ├── prod.tfvars
    │       └── staging.tfvars
    └── project2
        ├── main.tf
        ├── Makefile
        ├── outputs.tf
        ├── variables.tf
        └── vars
            ├── dev.tfvars
            ├── prod.tfvars
            └── staging.tfvars
```

## Continuous Deployment
This guide has assumed, so far, that individual team members are running Terraform to update infrastructure. However, when your Terraform deployment reaches a certain level of sophistication or your team reaches a size where it's becoming hard to keep track of which changes have been deployed to which environment, you might start to consider running Terraform with a CI/CD solution.

There are numerous ways to implement this but my recommendation would be to separate the Dockerfile into it's own repository and version it separately to your Terraform code.

Then, as part of your pipeline's deployment phase, you could run something like the following from within a project root module:
```
$ docker run --rm -it -v .:/terraform -e AWS_PROFILE=$AWS_PROFILE myorg/terraform:v0.0.4 make plan
$ docker run --rm -it -v .:/terraform -e AWS_PROFILE=$AWS_PROFILE myorg/terraform:v0.0.4 make apply
```
This mounts the project dir in the container and runs `plan` and `apply`. It also passes the `AWS_PROFILE` environment variable from your CI environment to the container. There's obviously endless bespoke configuration you could add here.

## Conclusion
This post gives you one approach to running terraform consistently that I've used with great success in a number of different organisations. I'd love to hear from anyone doing anything similar, or dissimilar, to achieve the same ends.
