## Reference:

- https://viblo.asia/p/thuc-hanh-set-up-kubernetes-cluster-tren-amazon-web-services-elastic-kubernetes-service-Qbq5QQEz5D8
- https://itnext.io/continuous-deployment-to-kubernetes-eks-using-aws-codepipeline-aws-codecommit-and-aws-codebuild-fce7d6c18e83

# 1 - Tạo EKS Cluster và deploy thử ứng dụng

### Install AWS CLI

Để configure AWS CLI, thì bạn sẽ cần phải chuẩn bị:

- Access Key ID
- Secret Access Key
- IAM Permission

### Hướng dẫ tạo IAM User và áp dụng giới hạn quyền lên User

- Đăng nhập vào IAM Management Console
- Tại thanh bên trái chọn Users rồi chọn Add user.
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/01.png">
- Tại trang Set user details, nhập những thông số sau rồi chọn Next Permissions:
  - User name: ec2-admin.
  - Access type: chọn Access key - Programmatic access
    <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/06.png">
- Tại mục Set permissions, bạn cần làm những thao tác sau:
  - Chọn Attach existing policies directly để gán policy trực tiếp vào IAM user.
  - Tìm và tick vào AdministratorAccess để gán quyền Admin cho IAM user.
    <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/02.png">
- Kiểm tra và chọn Next: Tags
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/03.png">
- Chọn Next Review để xem lại các thiết lập đã chọn:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/04.png">
- Tại trang Review, kiểm tra kỹ và chọn Create user.
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/05.png">
- Như vậy User sẽ được tạo và chúng ta sẽ có được Access Key và Secret Access Key để sử dụng. Bạn có thể tải tập tin .csv để lưu trữ thông tin Access Key.

### AWS configure - Kiểm tra thông tin biến môi trường của AWS trong Terminal của bạn

> aws configure

- Tại đây ta sẽ dùng cập Access Key vừa tạo để thiết lập môi trường kết nối với AWS và thực hiện cài đặt eksctl
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/07.png">

### Install eksctl

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

mv /tmp/eksctl /usr/local/bin

eksctl version
```

### Tạo k8s cluster trên AWS

- Trước tiên để có thể làm việc với EKS Cluster ta cần tạo key pair bằng dòng lệnh
  - Key Pair được dùng để mã hóa và giải mã thông tin đăng nhập vào máy chủ ảo EC2.

```
aws ec2 create-key-pair --key-name k8s-demo --query 'KeyMaterial' --output text> k8s-demo.pem
```

Hoặc có thể thao tác tạo key pair trên AWS Console

- Cà
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/08.png">

```
eksctl create cluster --name k8s-demo --region ap-southeast-1 --nodegroup-name k8s-demo --nodes 2 --ssh-access --ssh-public-key k8s-demo --managed
```

### Deploy mywebsite

```
kubectl create deployment mywebsite --image=minhtri2582/mywebsite

kubectl expose deployments/mywebsite --type=LoadBalancer --port=80

kubectl get svc
```

<br>

# 2 - Tạo CodePipeline CI/CD

### 2.1 - Tạo S3 bucket

```
ACCOUNT_ID=$(aws sts get-caller-identity | jq -r '.Account')
aws s3 mb s3://eks-${ACCOUNT_ID}-codepipeline-artifacts
```

### 2.2 - Tạo Role CodePipelineServiceRole

```
wget https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/cpAssumeRolePolicyDocument.json

aws iam create-role --role-name eks-CodePipelineServiceRole --assume-role-policy-document file://cpAssumeRolePolicyDocument.json

wget https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/cpPolicyDocument.json

aws iam put-role-policy --role-name eks-CodePipelineServiceRole --policy-name codepipeline-access --policy-document file://cpPolicyDocument.json
```

### 2.3 - Tạo Role CodeBuildServiceRole

```
wget https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/cbAssumeRolePolicyDocument.json

aws iam create-role --role-name eks-CodeBuildServiceRole --assume-role-policy-document file://cbAssumeRolePolicyDocument.json

wget https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/cbPolicyDocument.json

aws iam put-role-policy --role-name eks-CodeBuildServiceRole --policy-name codebuild-access --policy-document file://cbPolicyDocument.json
```

### 2.4 - Cho phép Role CodeBuildServiceRole có quyền cấu hình EKS cluster

- Từ terminal máy của bạn, lấy thông tin ConfigMap aws-auth

```
kubectl get configmaps aws-auth -n kube-system -o yaml > aws-auth.yaml
```

<br>

- Chỉnh file aws-auth.yaml, thêm role trong mục <b>data.mapRoles</b>

```
- groups:
        - system:masters
      rolearn: arn:aws:iam::{$AWS_ACCOUNT_ID}}:role/eks-CodeBuildServiceRole
      username: codebuild-eks
```

> Thay thế AWS_ACCOUNT_ID tương ứng

<br>

- File aws-auth.yaml sau khi thêm role <b>eks-CodeBuildServiceRole</b> có dạng như sau (Lưu ý: xóa dòng <b>metadata.creationTimestamp</b>)

```
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::435147975610:role/eksctl-k8s-demo-nodegroup-k8s-dem-NodeInstanceRole-189W51QXXFKU6
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
        - system:masters
      rolearn: arn:aws:iam::435147975610:role/eks-CodeBuildServiceRole
      username: codebuild-eks
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
  uid: 8e6402d2-383d-40f9-a0ba-8d6003ae5e4f
```

<br>

- Apply file aws-auth.yaml từ terminal của bạn

```
kubectl apply -f aws-auth.yaml
```

Kết quả cấu hình đã được apply:

```
# configmap/aws-auth unchanged configured
```

### 2.5 - Tạo CodePipeline bằng CloudFormation

1. Download template CloudFormation tại: https://raw.githubusercontent.com/minhtri2582/server-samples/3fd9f41672b171483db7ee495c834507991125ad/aws-journey/code_pipeline.yml

2. Vào CloudFormation - New Task

- Chọn file Upload: code_pipeline.yml
  <img src="https://raw.githubusercontent.com/minhtri2582/server-samples/master/CF-CreateTask.png"/>

- Điền thông tin Github và EKS Cluster đã tạo ở mục 1
  <img src="https://raw.githubusercontent.com/minhtri2582/server-samples/master/CF-Input.png"/>

- Chọn Create Task, chờ khoảng 5 phút để CloudFormation tạo các resource CodePipeLine và ECR Repository
  <img src="https://github.com/minhtri2582/server-samples/raw/master/CF-Progress.png"/>

- Tại AWS Console, vào trang quản trị CodePipeline, chọn Pipelines. Bạn sẽ thấy Pipeline đầu tiên đang chạy
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/CP-List.png"/>

- Click vào name Pipeline để xem chi tiết
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/CP-Details.png"/>

- Chờ đến khi Pipeline chạy thành công, click mục build - Details để xem CodeBuild Job
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/CP-Success.png">

- Màn hinh CodeBuild có thể xem log output trong quá trình build và deploy
  <img src="https://github.com/minhtri2582/server-samples/raw/master/CB-DetailSuccess.png"/>

### 2.5 - Kiểm tra CI/CD

- Chúng ta thử thay đổi nội dung thẻ title của file <b>index.htlm</b>

```
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Test CodePipeline</title>
```

- Click Commit changes trên Github
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/github-commit-index.png"/>

- Sau khi push code, vào trang quản trị CodePipeline sẽ thấy status In Progress
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/CP-trigger.png"/>

- Sau khi build Success, vào website xem thay đổi
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/web-change.png"/>

<br>

# Chúc mừng bạn đã hoàn thành bài lab!
