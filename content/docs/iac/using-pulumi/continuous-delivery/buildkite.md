---
title_tag: "Using Buildkite | CI/CD"
meta_desc: This page details how to use Buildkite pipelines to deploy infrastructure implemented using Pulumi.
title: Buildkite
h1: Buildkite
meta_image: /images/docs/meta-images/docs-meta.png
menu:
    iac:
        name: Buildkite
        parent: iac-using-pulumi-cicd
        weight: 2
---

[Buildkite](https://buildkite.com/) is a continuous delivery platform designed to scale with your projects.
Buildkite [powers](https://buildkite.com/platform/) the most demanding and sophisticated software companies in the world.

This guide shows you how you can run Pulumi in Buildkite, so you can go from deploying your cloud infrastructure to deploying
applications that run on that cloud infrastructure all in the same CI/CD service.

## Prerequisites

- Sign up for a [Pulumi account](https://app.pulumi.com)
- Create a [Pulumi Access Token](https://app.pulumi.com/account/tokens)
  - Keep this token somewhere safe. You'll need this later.
- Install the [latest Pulumi CLI](/docs/install/), so you can run the project locally as a test
- Create a new GitHub [repository](https://support.atlassian.com/bitbucket-cloud/docs/create-a-git-repository/)

- Create a [new Pulumi project](/tutorials/pulumi-fundamentals/create-a-pulumi-project/) and [initialize it as a git repository](https://git-scm.com/docs/git-init)

### Buildkite Agent

Buildkite offers two ways to run an agent. An agent is required to run your pipelines.
Agents can either be [managed](https://buildkite.com/docs/pipelines/getting-started#set-up-an-agent-create-a-buildkite-hosted-agent) (Buildkite hosted) or [self-hosted](https://buildkite.com/docs/pipelines/getting-started#set-up-an-agent-install-and-run-a-self-hosted-agent).
The [Pipelines architecture](https://buildkite.com/docs/pipelines/architecture) page has more information you might find helpful.

If you are kicking tires and want to get started quickly to explore all the features Buildkite has to offer,
we recommend that you [signup](https://buildkite.com/signup) for a hosted agent.

## Pipelines

With your Buildkite account and agent setup, you can configure pipelines using YAML.
You may use Buildkite's UI to create a pipeline quickly for a test too, but note that
YAML pipelines will soon replace the UI method of creating pipelines.

### Cloud Provider Setup

In this guide, we'll use AWS as the target cloud provider but the steps
are easily transferrable to other cloud providers.
Buildkite supports [OIDC auth with AWS](https://buildkite.com/docs/pipelines/security/oidc/aws).

Note the ARN for the AWS IAM role with permission to deploy to your AWS account.

### Pulumi Account Authentication

Pulumi CLI requires an access token to be able to store the infrastructure state
in your Pulumi Cloud account. The access token you created in the prerequisites
section should be saved as a [pipeline secret](https://buildkite.com/docs/pipelines/security/secrets/buildkite-secrets).

The token can also be stored in your cloud provider's own secret management service
or another secret management provider. See the [Managing Secrets](https://buildkite.com/docs/pipelines/security/secrets/managing) guide on Buildkite
to choose the method that best fits you as a longer-term solution.

### `.buildkite` Folder

* In your source repository, create the `.buildkite` folder in the root.
YAML pipelines must be created in this folder.
* Create a new file called `pipeline.yml` and paste the following pipeline configuration into the file.

**Note**: The pipeline assumes you are using the Node.js Pulumi SDK for your cloud infrastructure
but you can, of course, use any of the languages supported by Pulumi.

```yaml
env:
  AWS_ROLE_ARN: arn:aws:iam::AWS-ACCOUNT-ID:role/SOME-ROLE
  PULUMI_STACK: xxx

steps:
  - label: ":pulumi: Preview"
    commands:
      - npm install
      - pulumi preview -s $PULUMI_STACK | tee preview
      - printf '```\n%b\n```\n' "$(cat preview)" | buildkite-agent annotate --style "info"
    plugins:
      - aws-assume-role-with-web-identity#v1.0.0:
          role-arn: $AWS_ROLE_ARN

      - setup-pulumi:
        # Optional: The specific version to install
        # if you don't want to use the latest
        # available version.
        #
        # version: 3.183.0
```

Use of the `setup-pulumi` [Buildkite plugin for Pulumi](https://github.com/praneetloke/setup-pulumi-buildkite-plugin) is optional.
You may also consider using one of Pulumi's own [container images](https://github.com/pulumi/pulumi-docker-containers) that is specific to the language
you're using for your cloud infrastructure app. This version of the `pipeline.yml` file shows how you can use
one of those images (or even a custom image.)

```yaml
env:
  AWS_ROLE_ARN: arn:aws:iam::AWS-ACCOUNT-ID:role/SOME-ROLE
  PULUMI_STACK: xxx

steps:
  - label: ":pulumi: Preview"
    commands:
      - npm install
      - pulumi preview -s $PULUMI_STACK | tee preview
      - printf '```\n%b\n```\n' "$(cat preview)" | buildkite-agent annotate --style "info"
    plugins:
      - aws-assume-role-with-web-identity#v1.0.0:
          role-arn: $AWS_ROLE_ARN

      - docker#v5.9.0:
          image: "pulumi/pulumi-nodejs"
          propagate-aws-auth-tokens: true
          mount-buildkite-agent: true
          environment:
            - PULUMI_ACCESS_TOKEN
```

### Quickstart Template

Buildkite has a [CI/CD template](https://buildkite.com/platform/pipelines/templates/iac/pulumi-aws/) for creating AWS infrastructure using Pulumi.
Although the template assumes no CI triggers are used, it could be a good way to test things out.

