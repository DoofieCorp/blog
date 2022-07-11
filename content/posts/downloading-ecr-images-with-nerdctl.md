+++
title = "Downloading ECR images with nerdctl"
description = "Tutorial on how to use nerdctl to download private images from ECR"
keywords = ["docker", "ecr", "nerdctl", "images"]
categories = ["cloud", "aws", "docker", "containers"]
date = "2021-09-25T8:00:00Z"
+++

With Docker Desktop changing their licencing many folks are in need of a replacement. [Nerdctl](https://github.com/containerd/nerdctl) is worthy of this, combined with [Lima](https://github.com/lima-vm/lima) it provides a Docker like CLI interface.

While the CLI interface mostly remains the same, a question that might arise is "How to I pull/push images from private repositories?"

Lima provides a linux virtual environment for nerdctl to run in. By introducing a `$HOME/.docker/config.json` authentication can occur as normal.

By default, Lima will mount your home folder to `/Users/username` while maintaining its own home folder at `/home/username.linux`, for ease of configuration a be symbolic linked can be created  between the two:

```bash
osx $ lima
lima $ ln -s /Users/username/.docker $HOME/.docker
lima $ ln -s /Users/username/.aws $HOME/.aws
```

Doing this will make both your docker and AWS configurations available to nerdctl.

Assuming your docker configuration already contains a `credHelpers`:

```json
{
  "credHelpers": {
    "AWS_ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com": "ecr-login",
    "public.ecr.aws": "ecr-login"
  }
}
```

All thats left is to download [Amazon ECR Docker Credential Helper](https://github.com/awslabs/amazon-ecr-credential-helper) to somewhere on your $PATH within lima:

```bash
lima $ sudo wget -O /usr/bin/docker-credential-ecr-login https://amazon-ecr-credential-helper-releases.s3.us-east-2.amazonaws.com/0.5.0/linux-amd64/docker-credential-ecr-login 
lima $ sudo chmod 755 /usr/bin/docker-credential-ecr-login
```

You can now `nerdctl image pull <private-ecr-image>`.