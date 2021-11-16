# ECS Anywhere Hands-on Lab

ECS Anywhere 是 Amazon ECS 的扩展，允许客户在任何环境中部署本地 Amazon ECS 任务。 ECS Anywhere 依托 Amazon ECS 的易用性和简便性，在基于容器的应用程序之间提供一致的工具和 API 体验。无论是在本地还是云端，都可以实现相似的集群管理、工作负载计划并监控从 Amazon ECS 获得的内容。利用 ECS Anywhere 提供的完全托管解决方案，降低成本并减少复杂的容器编排。ECS Anywhere 可帮助您满足合规性要求并扩展业务规模，而不必牺牲本地投资。

本实验将指导您搭建一套ECS Anywhere环境，让您了解ECS Anywhere的工作原理，熟悉注册本地容器实例的步骤，了解如何从云端控制平面统一下发容器任务至本地计算环境，以及如何与云端服务联动充分发挥混合云的优势。

为保证实验过程的简便一致，本实验流程将使用EC2模拟外部实例的方式，除EC2启动过程外，其它操作步骤与使用本地计算实例完全一致。

## 实验步骤

### **创建EC2实例 (模拟外部实例)**

1. 在Cloud9命令行窗口中运行如下命令创建SSH Key、Security Group以及EC2实例
```
aws ec2 create-key-pair --key-name MyKeyPair --query 'KeyMaterial' --output text > MyKeyPair.pem
chmod 400 MyKeyPair.pem
SECURITY_GROUP_ID=`aws ec2 create-security-group --group-name ec2-ssh --description "SG for SSH login to EC2 instance" --output text`
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --to-port 22 --ip-protocol tcp --cidr-ip 0.0.0.0/0 --from-port 22
INSTANCE_ID=`aws ec2 run-instances --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 --instance-type t3.medium --key-name MyKeyPair --security-group-ids ${SECURITY_GROUP_ID} --query 'Instances[].InstanceId' --output text`
```

2. 等待约一分钟后，禁用该实例的IMDS，SSH登录该实例并安装epel
```
aws ec2 modify-instance-metadata-options --instance-id ${INSTANCE_ID} --http-endpoint disabled
INSTANCE_DNS_NAME=`aws ec2 describe-instances --instance-ids ${INSTANCE_ID} --query 'Reservations[].Instances[].PublicDnsName' --output text`
ssh -i MyKeyPair.pem -o StrictHostKeyChecking=no ec2-user@${INSTANCE_DNS_NAME}
sudo amazon-linux-extras install epel -y
```

### 创建ECS集群并注册外部实例

1. 在新的Cloud9命令行窗口中运行如下命令创建ECS集群
```
aws ecs create-cluster --cluster-name ECS-A-Test
```
2. 在浏览器窗口中打开https://ap-southeast-1.console.aws.amazon.com/ecs/home?region=ap-southeast-1#/clusters/ECS-A-Test/containerInstances，点击Register External Instances
3. 在Step 1: External instance activation details保持默认参数，点击Next step
4. 在Step 2: Register external instances点击Copy按钮以复制注册命令
5. 回到Cloud9已经SSH登录新建实例的命令行窗口，运行sudo su，然后粘贴并运行上面步骤中复制的命令
6. 如果一切正常，将会显示如下信息：
```
...
...
##########################
This script installed three open source packages that all use Apache License 2.0.
You can view their license information here:
  - ECS Agent https://github.com/aws/amazon-ecs-agent/blob/master/LICENSE
  - SSM Agent https://github.com/aws/amazon-ssm-agent/blob/master/LICENSE
  - Docker engine https://github.com/moby/moby/blob/master/LICENSE
##########################
```
7. 再回到ECS控制台，将会看到该实例已经注册到ECS Instances中

### 部署应用到外部容器实例上运行

1. 切换到Cloud9的命令行窗口，运行如下命令创建ECS Task definition：
```
cat <<EOF | xargs -0 aws ecs register-task-definition --cli-input-json
{
  "requiresCompatibilities": [
    "EXTERNAL"
  ],
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "public.ecr.aws/nginx/nginx:latest",
      "memory": 256,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 8080,
          "protocol": "tcp"
        }
      ]
    }
  ],
  "networkMode": "bridge",
  "family": "nginx"
}
EOF
```
2. 运行task:
```
aws ecs run-task --cluster ECS-A-Test --launch-type EXTERNAL --task-definition nginx
```
3. 回到外部实例(EC2)的SSH session内，检查nginx是否已经启动：
```
$ curl http://localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

