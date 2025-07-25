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
section should be saved as a [pipeline secret](https://buildkite.com/docs/pipelines/security/secrets/buildkite-secrets)
named `PULUMI_ACCESS_TOKEN`.

The token can also be stored in your cloud provider's own secret management service
or another secret management provider. See the [Managing Secrets](https://buildkite.com/docs/pipelines/security/secrets/managing) guide on Buildkite
to choose the method that best fits you as a longer-term solution.

As a long-term strategy, you should consider using OIDC with Pulumi, which allows Buildkite
to exchange an ID token with Pulumi for a short-lived Pulumi Access Token dynamically.

{{% notes type="info" %}}
Using OIDC auth with Pulumi Cloud is optional but encouraged.
{{% /notes %}}

Under your Pulumi organization's settings, click on **OIDC issuers** and then register a new issuer with
the following settings:

1. Issuer URL: `https://agent.buildkite.com`
1. Thumbprint: Leave blank. Pulumi will use the thumbprint of the server certificate for `https://agent.buildkite.com/.well-known/openid-configuration`
1. Click **Create issuer**

Once the issuer is created, the policy editor page will open. Update the settings to the following values:

1. Decision: Allow
1. Token type: [value is dependent on your Pulumi pricing tier]
1. Rules > `aud` claim: `urn:pulumi:org:{your Pulumi account name}` (Pulumi account name can be your individual account or your Pulumi org name that you see in the URL address bar.)
1. Rules > `sub` claim: See the format of the value used by Buildkite tokens: https://buildkite.com/docs/agent/v3/cli-oidc#claims.
    - If there are parts of `sub` string that you don't want to specify a value for, you must use a wildcard char `*` in its place. For example, if the organization name is `myorg` and the pipeline name is `mypipeline`, a `sub` claim value of `organization:myorg:pipeline:mypipeline:ref:*:commit:*:step:*` would mean that Pulumi would ignore the value of `ref`, `commit` and `step` tokens.
1. Add more claims if you would like Pulumi to validate additional claims in the Buildkite ID token.

See the Pulumi docs for [registering an OIDC issuer](/docs/pulumi-cloud/access-management/oidc-client/) for more information.

### `.buildkite` Folder

1. In your source repository, create the `.buildkite` folder in the root.
YAML pipelines must be created in this folder.
1. Create a new file called `pipeline.yml` and paste the following pipeline configuration into the file.

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
      - secrets:
        variables:
          # Map the PULUMI_ACCESS_TOKEN secret to an env var of the same name.
          # If you are using OIDC, this won't be needed.
          PULUMI_ACCESS_TOKEN: PULUMI_ACCESS_TOKEN

      - aws-assume-role-with-web-identity#v1.0.0:
          role-arn: $AWS_ROLE_ARN

      - setup-pulumi:
        # Optional: The specific version to install
        # if you don't want to use the latest
        # available version.
        #
        # version: 3.183.0
        #
        # Optional: Use self-managed backend URL instead of (default) Pulumi Cloud.
        # backend-url: s3://bucket_name/project_name/stack_name
        #
        # Optional: Use OIDC auth with Pulumi Cloud.
        # use-oidc: true
        # audience: urn:pulumi:org:{INDIVIDUAL_OR_ORG_LOGIN_NAME}
        # pulumi-token-type: urn:pulumi:token-type:access_token:personal
        # pulumi-token-scope: "user:{USER_LOGIN}"
```

Use of the `setup-pulumi` [Buildkite plugin for Pulumi](https://github.com/praneetloke/setup-pulumi-buildkite-plugin) is optional.
However, if you are using OIDC auth with Pulumi, you might consider using the plugin since it handles the OIDC token exchange
with Pulumi Cloud.

You may also consider using one of Pulumi's own container [images](https://github.com/pulumi/pulumi-docker-containers) for the language
you're using. Pulumi container images have the Pulumi CLI pre-installed along with the language runtime needed to run your programs.
The following example shows how you can use one of those images (or even a custom one.)

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
      - secrets:
          variables:
            # Map the PULUMI_ACCESS_TOKEN secret to an env var of the same name.
            # If you are using OIDC, this won't be needed.
            PULUMI_ACCESS_TOKEN: PULUMI_ACCESS_TOKEN

      - aws-assume-role-with-web-identity#v1.0.0:
          role-arn: $AWS_ROLE_ARN

      - docker#v5.9.0:
          image: "pulumi/pulumi-nodejs"
          propagate-aws-auth-tokens: true
          mount-buildkite-agent: true
          environment:
            - PULUMI_ACCESS_TOKEN
```

Create a pull request trigger by editing the [GitHub settings](https://buildkite.com/docs/pipelines/source-control/github#running-builds-on-pull-requests) in the pipeline.
See Buildkite docs on [source control](https://buildkite.com/docs/pipelines/source-control) for other VCS.

### Push trigger

Similarly, create a pipeline config YAML that runs `pulumi up` when a commit is pushed to your default branch.

```yaml
env:
  AWS_ROLE_ARN: arn:aws:iam::AWS-ACCOUNT-ID:role/SOME-ROLE
  PULUMI_STACK: xxx

steps:
  - label: ":pulumi: Preview"
    commands:
      - npm install
      - pulumi up -s $PULUMI_STACK
    plugins:
      - aws-assume-role-with-web-identity#v1.0.0:
          role-arn: $AWS_ROLE_ARN

      # Use setup-pulumi plugin or the Docker plugin to ensure Pulumi is installed.
      ...
```

### Quickstart Template

Buildkite has a [CI/CD template](https://buildkite.com/platform/pipelines/templates/iac/pulumi-aws/) for creating AWS infrastructure using Pulumi.

{{% notes type="info" %}}
Although the CI/CD template does not use any CI triggers, it could be another way to quickly test things out.
{{% /notes %}}

## Next Steps

This guide covered basic steps of running Pulumi in a Buildkite pipeline.
As your Pulumi project grows and you add more stacks to a project, based on your
strategy for mapping stacks to branches, especially, if you are using a stack to
represent a specific environment like test, staging or production.
You can make the stack name dynamic and run a pipeline for a specific stack
based on the branch.

### Dynamic Pipelines

While note strictly necessary, you can also create [dynamic pipelines](https://buildkite.com/docs/pipelines/configure/dynamic-pipelines).
Similar to Pulumi's concept of using higher-level programming languages for your cloud infrastructure,
Buildkite, too, allows you to [use a programming language](https://buildkite.com/docs/pipelines/configure/dynamic-pipelines/sdk)
to generate pipeline configurations dynamically at build time.

### Cache Volumes

[Cache volumes](https://buildkite.com/docs/pipelines/hosted-agents/cache-volumes) are external volumes attached to hosted agent instances.
They can be used to cache Pulumi plugins that get installed in `$HOME/.pulumi/plugins`, as well as any
language-specific packages, i.e. `node_modules` etc. Pulumi plugin binary versions are 1:1 with the
versions of the Pulumi packages that use them.
