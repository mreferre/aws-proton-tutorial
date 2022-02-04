### Deploying a generic container-based application using Proton 

This repository includes a tutorial that will guide users interested in Proton to appreciate the end-to-end workflows to deploy a working Proton service. As alluded in the introduction, there are many options available in Proton. In addition to, obviously, an infinite spectrum of use cases and templates that you can author, there are options in the IaC you can use, how you upload your templates to Proton, how you render and deploy those templates, whether or not you want an embedded and managed application deployment pipeline and possibly more. A tutorial cannot cover all the possible combinations so we will take an opinionated approach to demonstrate these end-to-end workflows. 

These are some of the decisions (and opinions) we took in the tutorial: 

- we will be using CloudFormation as the IaC of choice for this exercise 
- we will import the templates using the Git Sync method (as opposed to using the S3 upload mechanism)
- we will deploy a service with an embedded pipeline
- we will use a simple container-based application to focus on the Dev/Ops and CI/CD workflows in Proton
- we will mostly use the console in this tutorial (needless to say everything can be done )

> Also note that we need to strike a balance: the tutorial needs to be as simple as possible (so not to spend too much time on it) but, at the same time, sophisticated enough to demonstrate a typical Proton workflow. Do not expect the "time-to-wow" of Amazon Proton to be any close to the "time-to-wow" of [AWS App Runner](https://aws.amazon.com/apprunner/). They are two different services, with two different goals and objectives. 

This is what we are going to implement in this tutorial: 

![environments-services-basics](images/tutorial-diagram.png)

#### Prerequisites and housekeeping

The usual suspects. At a minimum: an AWS account, a working IDE environment with the AWS CLI, a GitHub account and a git tool of your choice.

You will need to fork this repository in your git account to start with. 

In addition, please fork [this simple container-based application](https://github.com/mreferre/nginx-custom-site). This will be the application code we will deploy with Proton. The peculiarity of this “application” is that it can be configured with a system variable to drive particular behaviors. Specifically, when the `INDEX_HTML_CONTENT` variable is passed to the container a custom web page with the content of the variable is shown with a specific hard-coded background. Please refer to the short README (https://github.com/mreferre/nginx-custom-site) file for more information. We will use the `INDEX_HTML_CONTENT` variable to simulate an application that uses different end-points in different environments (e.g. a test database or the production database). And we will use the hard-coded background to similute an application code change (with subsequent push to git and rebuild of the image). 

Last but not least, before we switch to the Proton console, you need to setup an AWS CodeStar connection following [these instructions](https://docs.aws.amazon.com/proton/latest/adminguide/setting-up-for-service.html#setting-up-vcontrol). This will allow you to access your GitHub account (and your repos) from Proton.

In an effort to simplify the tutorial, we are going to use an IAM administrative user/role to impersonate both a platform team admin as well as a developer. As a stretch goal, if you want, you may want to create two separate users with two separate AWS managed IAM policies (`AWSProtonFullAccess` and `AWSProtonDeveloperAccess`). In a production environment this would obviously be a best practice. 

#### Importing the templates 

This repository includes both the `environment` and the `service` templates (respectively in the [vpc-ecscluster-env](./vpc-ecscluster-env) folder and in the [lb-fargate-svc](./lb-fargate-svc) folder). You can have a look at the structure there and learn more about the layout in the [Template bundles](https://docs.aws.amazon.com/proton/latest/adminguide/ag-template-bundles.html) section of the Proton documentation. 

Go to the Proton console and switch to the `Settings/Repositories" page. Add the repository in your account that represents the fork of this repo. 

Now switch to the `Templates/Environment templates` page and click `Create environment template`. In the `Template bundle source` select `Sync templates from Git`. Pick your fork, set the Branch name to `main`. 

In the `Template details` set `vpc-ecscluster-env` as the `Template name`. 

> It is important that you set the name exactly because Proton will scan the repo for that exact folder name (and version structure). 

Set a `Template display name` for convenience, leave everything else as default and click `Create environment template`. 

Within seconds you should be seing 

