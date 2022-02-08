### Maintaining templates and deployments in Proton  

In the previous chapter we discovered how a Proton admin could import templates and how Proton admin and developers can deploy environments and services. This is table stake for Proton but not where Proton shines. You start appreciating the value of Proton when you start considering how admins can maintain the deployments for day2 operations. This is what we will explore in this chapter. 

> Important: a certain level of familiarity with editing CloudFormation templates is required for this chapter. 

### Updating the environment template [ PLATFORM ADMIN ]

Let's imagine that the platform administrator team has a new compliance requirement for VPC deployments that necessitates enabling `VPC flow log`. This was not enabled in the original environment templates. 

Open the CloudFormation template that represent the environment infrastructure and add the following YAML code right after the VPC and subnets definition: 

```
  VPCFlowLog:
    Type: AWS::EC2::FlowLog
    Properties:
      DeliverLogsPermissionArn: !GetAtt FlowLogRole.Arn
      LogGroupName: FlowLogsGroup
      ResourceId: !Ref VPC
      ResourceType: VPC
      TrafficType: ALL
  FlowLogRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Statement:
        - Effect: Allow
          Principal:
            Service: [ec2.amazonaws.com]
          Action: ['sts:AssumeRole']
      Path: /
      ManagedPolicyArns:
        - 'arn:aws:iam::aws:policy/CloudWatchLogsFullAccess'
  FlowLogsGroup: 
    Type: AWS::Logs::LogGroup
    Properties: 
      RetentionInDays: 7
```

This will create a CloudWatch LogGroup and will configure the VPC to use it as its flow log destination. Because this is not a breaking change to the VPC configuration we will update the template in the `v1` folder (this will result in a so called `minor version` change). Please read [this documentation page](https://docs.aws.amazon.com/proton/latest/adminguide/ag-template-versions.html) for a better understanding of minor and major versions. Commit the change to the repository. 

Because we have imported the template using `Git Sync`, Proton will discover the change in the template and will propose it as a `Draft`: 

![environment-template-minor-ver](../../images/environment-template-minor-ver.png)

> Note that each commit that modifies the template will automatically increase the minor version by one. In this case I made a few commits but I deleted the first two minor versions so Proton presents me version `1.3`. If this is your first commit you should see version `1.1` 

Go ahead and `Publish` the draft. Your `recommended` version should become `1.3`.

Note in the same environment template page you can see all environments (and their version) that have been originated from this template. This provides a good mechanism to track your deployments. 

### Updating the environments [ PLATFORM ADMIN ]

If you return to your `test` environment you will notice that under `Template version` it says `1.0` with a small information icon. If you expand that it will say that `you are not using the latest minor version.` 

You can then go ahead and `Update minor` (without changing anything in the wizard): 

![environment-minor-update](../../images/environment-minor-update.png)

At the end of this process the `Template version` should report the latest minor version and it should have a `recommended` green next to it (to remind the user that it's the latest available). If you are curious, in the EC2 console you can explore the two VPC and note that one (the `test` VPC) has the `Flow log` configuration filled properly while the other does not. 

You  Congratulations, you have just updated your first Proton environment by adding VPC flow log support. Go ahead and update the `production VPC as well`. 

### Updating the service template [ PLATFORM ADMIN ]

Now that we have updated both the environment template and the environments themselves, let's explore updating the services. Here is a situation that you, as a platform admin, may come across: you are getting requests from developers that they find it hard to debug their applications when they are running in the test environments (your policies do not allow you to enable exec'ing into containers in production but you can enable that for anything that is not production environments). Also, because of some incidents that have occurred over the last few weeks the business is requesting that all production deployments are gated by a manual approval from the business (this requires a change in the service pipeline). 

First locate the `pipeline_infrastructure` CloudFormation template, navigate to the section where the pipeline `Actions` are declared and add the following text snippet between the action named `Build` and the action named `'Deploy-{{service_instance.name}}'`: 
```
{% if 'production' in service_instance.name %}
        - Actions:
            - ActionTypeId:
                Category: Approval
                Owner: AWS
                Provider: Manual
                Version: '1'
              InputArtifacts: []
              Name: Approval
              RunOrder: 1
          Name: Preproduction_Approval
{% endif %}
``` 
There is quite a bit of Jinja magic going on here. Here is what's happening. You just placed an action inside a `for` loop (`{%- for service_instance in service_instances %}`) that basically create a Deploy action per each service instance you created. What you added above is a piece of code that, basically, says "if the instance_name is called `production` then add an pipeline action that is a manual approval. The developer knows that when they create an instance called `production` Proton will add a manual approval gate. 

> there is a bit of naming convention here that needs to be agreed between the developer and the platform team. The developers know that when they create an instance called `production` Proton will add a manual approval gate to the pipeline before deploying it. 

Now let's enable [ECS exec](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-exec.html) for the service itself (the following changes are required to enable the ECS Exec feature). 

Locate the `instance_infrastructure` CloudFormation template and add the following snippet somewhere in the `Resources` section: 

```
  ECSTaskRole:
      Type: AWS::IAM::Role
      Properties:
          AssumeRolePolicyDocument:
              Statement:
              - Effect: Allow
                Principal:
                   Service: [ecs-tasks.amazonaws.com]
                Action: ['sts:AssumeRole']
          Path: /
          Policies:
              - PolicyName: SSMMessagesPolicy
                PolicyDocument:
                  Statement:
                      - Effect: Allow
                        Action:
                          - 'ssmmessages:CreateControlChannel'
                          - 'ssmmessages:CreateDataChannel'
                          - 'ssmmessages:OpenControlChannel'
                          - 'ssmmessages:OpenDataChannel'
                        Resource: '*'
```

Locate the line that says `TaskRoleArn: !Ref "AWS::NoValue"` and change it to `TaskRoleArn: !Ref 'ECSTaskRole'`

Last but not list add the `{% if 'test' in service_instance.name %} EnableExecuteCommand: true {% endif %}` in the `Service` resource properties. It should look something like this when added (note how we are using Jinja to only the required parameter for the deployments of the `test` instances):  

```
  Service:
    Type: AWS::ECS::Service
    DependsOn: LoadBalancerRule
    Properties:
      {% if 'test' in service_instance.name %} EnableExecuteCommand: true {% endif %} 
      ServiceName: '{{service.name}}_{{service_instance.name}}'
```

> again, there is a bit of naming convention here that needs to be agreed between the developer and the platform team. The developers know that when they create a service instance called `test` Proton will add a parameter to the ECS tasks that will allow developers to exec into them for debugging purposes. Note that the developer doesn't need to know anything about how to make it happen. 

Now that you modified the service instance and pipeline properties, you can push the changes to GitHub. Proton should detect a new minor version that you can publish. That will become the new `recommended` version: 

![service-template-minor-ver](../../images/service-template-minor-ver.png)

The work of the platform admin team is done here! 

### Updating the service [ DEVELOPER ]

Now you are back as a developer. As you explore the service that you deployed, you notice something. Proton hints you that the administrator has made available a new (minor) version of the service template. You know this is going to introduce new operational improvements so you are eager to update your service to use the latest template. 

> note you can't update the service as a whole to the new minor template. You need to individually update each instance and the pipeline. 

We will start with the pipeline. Move to the `Pipeline` tab of your service, click the informational warning on your template and click `Update to recommended version`: 

![pipeline-update](../../images/pipeline-update.png)

Within a few minutes the pipeline should be updated. If you explore its layout you will note that now there is indeed a manual approval right before the deployment of the `production` instance:

![new-pipeline-with-approval](../../images/new-pipeline-with-approval.png)




