---
date: "2017-06-01T20:29:26+08:00"
title: "使用AWS ECS部署应用并自动扩展"
description: autoscaling with AWS ECS
---

最近在写一些 nodejs 的东西，尝试用 docker 去部署，研究了一下如何用 AWS ECS 做部署和自动扩展，跟传统的使用 EC2 来做部署使用 ASG 做扩展还是有一些不同的。

## AWS ECS 基本概念

Amazon EC2 Container Service 是基于 docker 的具有高可扩展性，快速的容器管理服务，可以让你在 EC2 instance 集群中简单快速的部署和扩展应用程序。

![](https://ws4.sinaimg.cn/large/006tKfTcly1fg78xsfn3jj315u0e60w0.jpg)

ECS 中有一些基本概念需要先了解一下

- ECR - Elastic Container Registry 用来建立一个安全，可扩展，高可用的应用程序私有 docker registry。
- Cluster - 逻辑上的一组由 EC2 instance 组成的集群，你的 container 跑在这些集群上面。
- Service - 在指定的 cluster 上来维护和启动一定数量的任务容器实例的服务。
- Task Definition - 定义了由 service 维护的任务，比如 image 是哪个，CPU 和 Memory 如何分配，端口号如何映射，容器跑起来时候执行哪些命令，有哪些环境变量设置等等。

## 如何使用 ECS 部署

接下来会介绍如何基于 Application Load Balancer 和 ECS 来部署自己的 application

### 1. 打包 docker image 并 push 到 ECR

我们从 Dockerfile 来打包一个 demo 的 image:

```dockerfile
FROM nodejs:7.6.0

COPY . /source

COPY ./env/production.env /source/.env

WORKDIR /source

RUN yarn install --production

EXPOSE 3000

CMD ["yarn", "start"]

```

执行 `docker build -t 12345678.dkr.ecr.ap-northeast-1.amazonaws.com/demo:0.0.1 .` 来打包 image

成功后 push 到 ECR 中

```shell
$ aws ecr get-login --region ap-southeast-1
$ docker push 12345678.dkr.ecr.ap-northeast-1.amazonaws.com/demo:0.0.1
```

可以自己写个 shell 脚本把这些都自动化起来

### 2. 使用 CloudFormation 建立一个 Application Load Balancer

可以参考 AWS CF 模板中 ALB 部分来写，主要是有 AWS::ElasticLoadBalancingV2::LoadBalancer， AWS::ElasticLoadBalancingV2::Listener，AWS::ElasticLoadBalancingV2::TargetGroup 这些资源，参考这个[demo 模板](https://github.com/liul85/Autoscale-with-AWS--ECS/blob/master/alb.yml)

### 3. 利用 CloudFormation 来建立一个 cluster

Cluster 就是一组运行的 EC2 instance，它需要在 autoscaling 组里面来管理保证 cluster 是可以随时扩展，所有这部分模板中应该保证有 LaunchConfiguration， Cluster，Role，SecurityGroup 参考这个[demo 的模板](https://github.com/liul85/Autoscale-with-AWS--ECS/blob/master/cluster.yml)

### 4. 建立一个 ECS service

有了 cluster 之后，我们就可以利用 service 把容器任务运行起来了，这里我们再建一个 CF 模板来创建 service 的资源，

要想用 service 来管理容器任务，首先定义一个 task，在 TD 里面我们定义了一个 container，它有 name，cpu，memoery 等一些基本属性，注意这里的 PortMappings，我们把 HostPort 设置为 0(或者也可以不设置)，这里是为了使用动态端口映射，这样当这个 container 运行起来之后，ECS 会给它随机分配一个主机端口，目前范围是从 32768~61000，这样我们就可以在一个 instance 上同时跑多个 container 了，具体可以参考[这里](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html)的 portMappings 中对 hostPort 的说明.

```yaml
TaskDefinition:
  Type: AWS::ECS::TaskDefinition
  Properties:
    ContainerDefinitions:
      - Name: !Join ["-", [!Ref "ServiceName", !Ref "BuildNumber"]]
        Cpu: 512
        Essential: "true"
        Memory: 512
        PortMappings:
          - HostPort: 0
            Protocol: tcp
            ContainerPort: !Ref "ContainerPort"
        Image:
          !Join [
            ":",
            [!Join ["/", [!Ref "ECR", !Ref "ServiceName"]], !Ref "BuildNumber"],
          ]
```

然后我们就可以创建一个 service 来使用这个 task definition，这个 service 会关联到之前创建的 LoadBalancer 中，我们使用的是 Application Load Balancer，需要在这里指定它关联的 TargetGroupArn，这样当容器任务跑起来之后，service 会把它注册到这个 targetGroup 下面。

```yaml
ECSService:
  Type: AWS::ECS::Service
  Properties:
    Cluster:
      Ref: ECSCluster
    DesiredCount:
      Ref: DesiredCount
    LoadBalancers:
      - ContainerName: !Join ["-", [!Ref "ServiceName", !Ref "BuildNumber"]]
        ContainerPort: 3000
        TargetGroupArn:
          Ref: TargetGroup
    DeploymentConfiguration:
      MaximumPercent: "200"
      MinimumHealthyPercent: "50"
    TaskDefinition:
      Ref: TaskDefinition
    Role:
      Ref: ECSRole
```

当这三个 CF 创建的资源全都成功后，你的 application 就已经在容器中运行了，你可以通过 ALB 的地址来访问它。

### 5. 部署新版本

当有新的提交之后，我们会有新的版本，就会打包新的 image 并 push 到 ECR 中，这个时候只需要把 service 中的 BuildNumber 修改为新版本，然后 update 这个 Stack，service 会自动创建新的 TaskDefinition 并运行起来，然后把老的 Task 从 targetGroup 中去掉，这里会有一个 draining 时间，默认 5 分钟，`MinimumHealthyPercent` 参数指定了在部署过程中原来的版本最小百分比，保证在部署过程中业务不会中断，这样就完成了新旧版本的替换。

## 自动扩展

上面我们的 application 已经可以运行在 ECS 服务中了，但是还没有任何自动扩展功能，当遇到很大的访问量时候就会有问题了，我们需要添加自动扩展，在 ECS 中自动扩展包括两部分

#### Cluster 扩展

Cluster 扩展主要是通过 EC2 的 autoScalingGroup 来完成的，我们定义了 ASG，scale up 和 scale down 的 policy，以及触发这些 policy 的 alarm 就可以完成了。

#### Service 扩展

service 的扩展我们需要建立 ServiceScalingTarget，它指定了最小和最大的 capacity，以及哪个 service 扩展

```yaml
ServiceScalingTarget:
  Type: AWS::ApplicationAutoScaling::ScalableTarget
  DependsOn: ECSService
  Properties:
    MaxCapacity: !Ref "ServiceMaxASG"
    MinCapacity: !Ref "ServiceMinASG"
    ResourceId:
      !Join ["", [service/, !Ref "ECSCluster", /, !GetAtt [ECSService, Name]]]
    RoleARN: !GetAtt [ECSAutoscalingRole, Arn]
    ScalableDimension: ecs:service:DesiredCount
    ServiceNamespace: ecs
```

然后需要创建 scale up policy，与 serviceScaleTarget 关联起来，当需要 scale up 的时候，会触发这个 policy，我们配置的 AdjustmentType 是`ChangeInCapacity`同时 stepAdjustments 里面配置了 ScalingAdjustment 是 1，即 policy 执行时候会增加一个容器任务做到横向 service 扩展，详细参考[这里](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-applicationautoscaling-scalingpolicy-stepscalingpolicyconfiguration.html#cfn-applicationautoscaling-scalingpolicy-stepscalingpolicyconfiguration-stepadjustments)。

```Yaml
ServiceScalingUpPolicy:
    Type: AWS::ApplicationAutoScaling::ScalingPolicy
    Properties:
      PolicyName: ScaleUpPolicy
      PolicyType: StepScaling
      ScalingTargetId: !Ref 'ServiceScalingTarget'
      StepScalingPolicyConfiguration:
        AdjustmentType: ChangeInCapacity
        Cooldown: 300
        MetricAggregationType: Average
        StepAdjustments:
        - MetricIntervalLowerBound: 0
          ScalingAdjustment: 1
```

需要一个 alarm 来触发这个 sale up policy，通过 cloudWatch 中 AWS/ECS 命名空间下面的以 ServiceName 为维度的指标来触发，当 CPU 占用率在 5 分钟内大于 60%即上报告警，触发 scale up 操作。

```yaml
ECSCPUHighAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    EvaluationPeriods: 5
    Statistic: Average
    Threshold: 60
    AlarmDescription: Alarm if CPU utilization if great than 60
    Period: 60
    AlarmActions:
      - Ref: ServiceScalingUpPolicy
    Namespace: AWS/ECS
    Dimensions:
      - Name: ServiceName
        Value:
          Ref: ECSService
    ComparisonOperator: GreaterThanThreshold
    MetricName: CPUUtilization
```

以上即是在 AWS 上使用 ECS 部署应用程序，并配置自动扩展策略的最简单实践。上面 3 个 CloudFormation 的模板可以参考[这里](https://github.com/liul85/Autoscale-with-AWS--ECS)

参考：

https://aws.amazon.com/blogs/compute/automatic-scaling-with-amazon-ecs/
