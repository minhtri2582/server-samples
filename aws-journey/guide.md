# Giới thiệu

Kubernetes (K8s) là một nền tảng nguồn mở, có thể mở rộng để quản lý các ứng dụng được đóng gói và các service, giúp thuận lợi trong việc cấu hình và tự động hoá việc triển khai ứng dụng

Trong bài lab này, chúng ta sẽ tạo một Kubernetes Cluster đơn giản trên AWS EKS (Elastic Kubernetes Service). Sau đó, mình sẽ triển khai trang web trên cluster và sử dụng AWS CodePipeLine để triển khai tự động. 

**Mô hình cơ bản EKS Cluster:**

<img src="https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/eks.png"/>

**Nội dung bài viết:**
1. Tạo EKS Cluster và triển khai ứng dụng
2. Tạo CodePipeline CI/CD
3. Kiểm tra CI/CD

# 1 - Tạo EKS Cluster và triển khai ứng dụng

### 1.1 - Install AWS CLI 
Để có thể sử dụng được  command  aws trên  terminal  (trên Linux/MAC) hoặc  cmd(Window) ta cần phải cài đặt AWS CLI.
Tham khảo thêm ở đây https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html.

Ở đây mình sử dụng cách đơn giản nhất cho MAC/Window là cài đặt trực tiếp file .exe(đối với Window) và file .pkg (đối với MAC).
>Lưu ý là guide này sử dụng cho terminal của MAC hoặc Linux.
- MAC: https://awscli.amazonaws.com/AWSCLIV2.pkg
- Window: https://awscli.amazonaws.com/AWSCLIV2.pkg
- Linux: mở terminal và dùng lệnh 
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" 
unzip awscliv2.zip
sudo ./aws/install 
```
Sau khi cài xong, mở terminal và gõ lệnh này vào 
``` 
aws --version
// Nếu terminal của bạn hiển thị như vầy 
aws-cli/2.4.0 Python/3.8.8 Darwin/20.3.0 exe/x86_64 prompt/off
```

>Cài đặt AWS CLI đã xong

### 1.2 - Cấu hình AWS profile
#### 1.2.1 - Tạo Access Key ID và Secret Access Key
Bước tiếp theo là cấu hình AWS CLI để có thể truy cập vào các tài nguyên của AWS.  

- Đăng nhập vào **IAM Management Console**
- Tại thanh bên trái chọn Users rồi chọn **Add user:**
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/01.png">

<br>

- Tại trang Set user details, nhập những thông số sau rồi chọn **Next Permissions**:
  - User name: **ec2-admin**
  - Access type: chọn **Access key - Programmatic access**
    <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/06.png">

<br>

- Tại mục **Set permissions**, bạn cần làm những thao tác sau:
  - Chọn **Attach existing policies directly** để gán policy trực tiếp vào IAM user.
  - Tìm và tick vào **AdministratorAccess** để gán quyền Admin cho IAM user. L
  >**Lưu ý**: Đối với môi trường production thì bạn không nên làm gán quyền admin như này mà phải giới hạn permission của từng User._
    
<img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/02.png">

<br>

- Kiểm tra và chọn **Next: Tags**
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/03.png">

<br>

- Chọn **Next Review** để xem lại các thiết lập đã chọn:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/04.png">

<br>

- Tại trang **Review**, kiểm tra kỹ và chọn **Create user**:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/05.png">
>_Như vậy User sẽ được tạo và chúng ta sẽ có được **Access Key** và **Secret Access Key** để sử dụng. Bạn có thể tải tập tin .csv để lưu trữ thông tin Access Key._

#### 1.2.3 - Thiết lập aws configure
- Từ teriminal, gõ command:
```shell
aws configure
``` 

- Tại đây ta sẽ dùng cập Access Key vừa tạo để thiết lập môi trường kết nối với AWS và thực hiện cài đặt eksctl:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/07.png">


### 1.3 - Install eksctl
Tương tự AWS CLI, **eksctl** là một CLI đơn giản để tạo và quản lý các  cluster trên EKS. 

Để sử dụng được eksctl, tại terminal sau khi đã config AWS account profile trong terminal, gõ lệnh sau để cài đặt: 
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
mv /tmp/eksctl /usr/local/bin
eksctl version
```

### 1.4 - Tạo EKS cluster trên AWS

- Để tạo EKS cluster, trước tiên chúng ta sẽ phải tạo SSH key để có thể access vào EC2 Node trong Cluster khi cần thiết:

```shell
aws ec2 create-key-pair --key-name k8s-demo --query 'KeyMaterial' --output text> k8s-demo.pem
```

- Hoặc có thể thao tác tạo key pair trên AWS Console:
  - Từ AWS Console vào mục **EC2**
  - Chọn mục **Key Pair** -> Chọn **Create key pair**
    <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/01.png">
  - Tại màn hình Create key pair nhập **Name** và chọn **Create key pair**
    <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/02.png">

- Cả 2 cách dùng **AWS Console** và **AWS CLI** đều cho ra kết quả như sau:
<img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/08.png">

- Để tạo EKS Cluster và các EC2 Node chúng ta sử dụng command sau:

```shell
eksctl create cluster --name k8s-demo --region ap-southeast-1 --nodegroup-name k8s-demo --nodes 2 --ssh-access --ssh-public-key k8s-demo --managed
```

- Khi bạn chạy command này, thì eksctl sẽ sử dụng **AWS CloudFormation** để tạo các infrastructure cần thiết và setup Master Node (Control Plane).
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/09.png">
- Trong quá trình chờ, Bạn có thể sử dụng giao diện quản lý AWS CloudFormation kiểm tra trạng thái.
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/03.png">
- Để kiểm tra, mình sẽ sử dụng kubectl:

```shell
kubectl get nodes
```

  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/10.png" alt="">

- Kiểm tra trên **AWS Console** để xem các Resource được tạo
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/11.png">
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/12.png">
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/13.png">

### 1.5 Deploy ứng dụng website

- Tiếp theo, mình sẽ deploy một static Website vào EKS Cluster mình vừa tạo. Để deploy, đầu tiên mình sẽ chạy command sau:

```sh
kubectl create deployment mywebsite --image=minhtri2582/mywebsite
```
>TODO: Hướng dẫn cài kubectl!!!!!

- Để có thể truy cập Website từ bên ngoài EKS cluster, chúng ta sẽ phải deploy một LoadBalancer Service vào cluster bằng command sau:

```sh
kubectl expose deployments/mywebsite --type=LoadBalancer --port=80
```
> **Lưu ý**: _tên deployment **mywebsite**, chúng ra sẽ dùng tên này trong phần 2, bạn có thể thay đổi tên deployment nhưng phải match với tên ở phần 2._ 

- Để xem thông tin về LoadBalancer trên, mình sẽ chạy command sau:

```sh
kubectl get svc
```

  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/14.png" alt="">

- Copy link trong External-ip vào trình duyệt sẽ truy cập được website:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/15.png">

  
<br>

# 2 - Tạo CI/CD với AWS CodePipeline
Đây là sơ đồ triển khai CI/CD bằng công cụ AWS CodePipeline chúng ta sẽ làm trong phần này:
<img src="https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/cicd.png"/>

### 2.1 - Chuẩn bị (S3, IAM Roles, RBAC) 
#### 2.1.1 - Tạo S3 bucket
Bucket này cho CodeBuild lưu artifact (tạo ra tập tin build.json sau mỗi lần build)
```shell
ACCOUNT_ID=$(aws sts get-caller-identity | jq -r '.Account')
aws s3 mb s3://eks-${ACCOUNT_ID}-codepipeline-artifacts
```

#### 2.1.2 - Tạo role eks-CodePipelineServiceRole

**CodePileline** và **CodeBuild** cần phải có IAM roles để build docker, push image và tương tác với EKS cluster bằng command **kubectl**.
Hãy tải các file json chính sách, tạo các role **eks-CodePipelineServiceRole, eks-CodePipelineServiceRole** và thêm inline policy từ terminal như sau:
```shell
wget https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/cpAssumeRolePolicyDocument.json

aws iam create-role --role-name eks-CodePipelineServiceRole --assume-role-policy-document file://cpAssumeRolePolicyDocument.json

wget https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/cpPolicyDocument.json

aws iam put-role-policy --role-name eks-CodePipelineServiceRole --policy-name codepipeline-access --policy-document file://cpPolicyDocument.json
```

#### 2.1.3 - Tạo role eks-CodeBuildServiceRole

```shell
wget https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/cbAssumeRolePolicyDocument.json

aws iam create-role --role-name eks-CodeBuildServiceRole --assume-role-policy-document file://cbAssumeRolePolicyDocument.json

wget https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/cbPolicyDocument.json

aws iam put-role-policy --role-name eks-CodeBuildServiceRole --policy-name codebuild-access --policy-document file://cbPolicyDocument.json
```
> Có thể kiểm tra role đã tạo, vào **AWS Console - IAM - Roles**
> <img src="https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/iam-roles.png" />

#### 2.1.4 -  Cho phép role eks-CodeBuildServiceRolequyền trong RBAC của EKS cluster
Để CodeBuild có thể tương tác với EKS, chúng ta sẽ chỉnh sửa configMap **aws-auth**
- Từ terminal đã kết nối EKS cluster, lấy bản sao configMap aws-auth bằng command:

```shell
kubectl get configmaps aws-auth -n kube-system -o yaml > aws-auth.yaml
```

- Tập tin này sẽ có định dang như sau:
```yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::435147975610:role/eksctl-k8s-demo-nodegroup-k8s-dem-NodeInstanceRole-189W51QXXFKU6
      username: system:node:{{EC2PrivateDNSName}}         
kind: ConfigMap
metadata:
  creationTimestamp: "2022-03-13T12:28:45Z"
  name: aws-auth
  namespace: kube-system
  resourceVersion: "1416"
  uid: 8e6402d2-383d-40f9-a0ba-8d6003ae5e4f
```

- Chỉnh file aws-auth.yaml, thêm một item trong mảng **data.mapRoles**:

```yaml
- groups:
    - system:masters
  rolearn: arn:aws:iam::{$AWS_ACCOUNT_ID}}:role/eks-CodeBuildServiceRole
  username: codebuild-eks
```
_(Thay thế **AWS_ACCOUNT_ID** tương ứng)_


- Tập tin **aws-auth.yaml** sau khi thêm role _eks-CodeBuildServiceRole_ có dạng như sau 
(**Lưu ý**: xóa dòng _metadata.creationTimestamp_ và _metadata.resourceVersion_)

```yaml
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

- Apply tập tin aws-auth.yaml đã thay đổi từ terminal:

```sh
kubectl apply -f aws-auth.yaml
```

- Sau khi chạy thành công màn hình có kết quả như sau: 

```
# configmap/aws-auth unchanged configured
```

### 2.2 - Fork source code về GitHub
```shell
https://github.com/minhtri2582/eks-web-example
```
> **TODO:** hướng dẫn fork source từ Github https://github.com/minhtri2582/eks-web-example để có quyền push code, tạo personal token

### 2.3 - Tạo CodePipeline bằng CloudFormation
Bước tiếp theo chúng ta sẽ tạo CodePipeline sử dụng công cụ AWS CloudFormation. 
#### 2.3.1 - Download template
Download CloudFormation template file tại:  
https://raw.githubusercontent.com/minhtri2582/server-samples/3fd9f41672b171483db7ee495c834507991125ad/aws-journey/code_pipeline.yml

<br>

#### 2.3.2 - Tạo stack
Trong màn hình quản trị CloudFormation: Chọn **Create stack - with new resources (standard)**

- Upload a template file: Chọn file **code_pipeline.yml** đã download ở bước 2.3.1. Click **Next**:
  <img src="https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/CF-CreateTask.png"/>

<br>

- Nhập thông tin GitHub (đã tạo ở phần 2.2) và EKS Cluster (đã tạo ở phần 1):
  <img src="https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/CF-Input.png"/>
  
  - Username: **[github account]**
  - Access token: **[token tạo bước trước]**
  - Repository: **[tên repository]**
  - Branch: **[nhánh của repository]**
  - EksCluster: **k8s-demo**
  - EksDeployment: **mywebsite** (phải cùng với tên k8s deployment ở phần 1)
  - EksNamespace: **default** (cùng với namespace đã tạo deployment ở phần 1, mặc định là default)

  Click **Next** - **Next** và **Create Stack** để tạo stack.



- Chờ khoảng 5 phút để CloudFormation tạo các resource _CodePipeLine_, _CodeBuild_ và _ECR Repository_:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/CF-Progress.png"/>

  <br>

- Vào trang quản trị **CodePipeline**, chọn Pipelines. Bạn sẽ thấy Pipeline đầu tiên đang chạy:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/CP-List.png"/>
  
<br>

- Click vào tên pipeline (**eks-codepipeline-CodePipelineGithubXXXXXXX**) để xem chi tiết:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/CP-Details.png"/>

<br>

- Chờ đến khi Pipeline chạy thành công. Trong stage **Build** - Click **Details** để xem CodeBuild Job:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/CP-Success.png">

<br>

- Chúng ta có thể xem log output trong quá trình build và deploy:
  <img src="https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/CB-DetailSuccess.png"/>

### 2.4 - Kiểm tra CI/CD
1. Vào trang https://github.com/, chọn repository đã fork ở phần 2.2:

> **TODO:** Chụp hình source code GitHub

<br>

2. Thử thay đổi nội dung thẻ title của tập tin <b>index.htlm</b>:

```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Test CodePipeline</title>
```
> **TODO:** Chụp hình màn hình edit file index.htlm

<br> 

3. Click **Commit changes**:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/github-commit-index.png"/>

<br>

4. Sau khi push code, vào trang quản trị **CodePipeline** sẽ thấy trạng thái của pipeline là _In Progress_:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/CP-trigger.png"/>

5. Chờ 5 phút để quá trình build _Success_. Vào website URL xem thay đổi:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/web-change.png"/>

<br>
