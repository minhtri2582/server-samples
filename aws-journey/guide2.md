# Giới thiệu

Kubernetes (K8s) là một nền tảng nguồn mở, có thể mở rộng để quản lý các ứng dụng được đóng gói và các service, giúp thuận lợi trong việc cấu hình và tự động hoá việc triển khai ứng dụng

Trong bài lab này, chúng ta sẽ tạo một Kubernetes Cluster đơn giản trên AWS EKS (Elastic Kubernetes Service). Sau đó, mình sẽ triển khai trang web trên cluster và sử dụng AWS CodePipeLine để triển khai tự động.

**Mô hình cơ bản EKS Cluster:**

<img src="https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/eks.png"/>

**Nội dung bài viết:**
1. Tạo EKS Cluster và triển khai một ứng dụng web
2. Tạo CodePipeline CI/CD
3. Kiểm tra CI/CD

# 1 - Tạo EKS Cluster và triển khai ứng dụng

### Install AWS CLI
Để có thể sử dụng được  command  aws trên  terminal  (trên Linux/MAC) hoặc  cmd(Window) ta cần phải cài đặt AWS CLI.
Tham khảo thêm ở đây https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html.
Ở đây mình sử dụng cách đơn giản nhất cho MAC/Window là cài đặt trực tiếp file .exe(đối với Window) và file .pkg (đối với MAC).
Lưu ý là guide này sử dụng cho terminal của MAC hoặc Linux.
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

Cài đặt AWS CLI đã xong

##### Bước tiếp theo là cấu hình AWS CLI để có thể access vào  account AWS
- IAM Permission
- Access Key ID
- Secret Access Key

### Tạo IAM User và áp dụng giới hạn quyền lên User

- Đăng nhập vào IAM Management Console
- Tại thanh bên trái chọn Users rồi chọn Add user.
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/01.png">
- Tại trang Set user details, nhập những thông số sau rồi chọn Next Permissions:
    - User name: ec2-admin.
    - Access type: chọn Access key - Programmatic access
      <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/add-user/06.png">
- Tại mục Set permissions, bạn cần làm những thao tác sau:
    - Chọn Attach existing policies directly để gán policy trực tiếp vào IAM user.
    - Tìm và tick vào AdministratorAccess để gán quyền Admin cho IAM user. Lưu ý: Đối với môi trường production thì bạn không nên làm gán quyền admin như này mà phải giới hạn permission của từng User.
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
Tương tự AWS CLI, eksctl là một CLI đơn giản để tạo và quản lý các  cluster trên EKS
Để sử dụng được eksctl, trước hết phải cài đặt, các bước cài đặt sử dụng  terminal sau khi đã config AWS account profile trong terminal, chúng ta chỉ cần mở terminal lên và gõ lệnh sau để cài đặt. Việc cài đặt này có thể thực thi ở bất cứ một device nào đã  config AWS profile (thậm chí là máy ảo)
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
mv /tmp/eksctl /usr/local/bin
eksctl version
```

### Tạo k8s cluster trên AWS

- Để tạo EKS cluster, trước tiên chúng ta sẽ phải tạo SSH key để có thể access vào EC2 Node trong Cluster khi cần thiết (Đây cũng là best practices mà Google khuyên các bạn làm theo).
    - Key Pair được dùng để mã hóa và giải mã thông tin đăng nhập vào máy chủ ảo EC2.
    - Command này sẽ tạo một KeyPair có tên là k8s-demo và sẽ lưu kết quả vào k8s-demo.pem file.

```
aws ec2 create-key-pair --key-name k8s-demo --query 'KeyMaterial' --output text> k8s-demo.pem
cat k8s-demo.pem
// Sau đó copy Private key lại để sau này sử dụng
```

Hoặc có thể thao tác tạo key pair trên AWS Console

- Từ AWS Console vào mục EC2
- Chọn mục Key Pair -> Chọn Create key pair
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/01.png">
- Tại màn hình Create key pair nhập Name và chọn Create key pair
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/02.png">

Cả 2 cách dùng AWS Console và AWS CLI đều cho ra kết quả như sau:
<img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/08.png">

- Để tạo EKS Cluster và các 3 Node loại Spot Instance (giá rẻ hơn nhiều), chúng ta sử dụng command sau:

```shell
$ eksctl create cluster --spot --instance-types=m3.large,m4.large,m5.large --nodes 3 --name k8s-demo --region ap-southeast-1 --nodegroup-name k8s-demo
$ eksctl create cluster --spot --instance-types=m3.large,m4.large,m5.large --nodes 3 --name k8s-demo --region ap-southeast-1 --nodegroup-name k8s-demo
```

- Khi bạn chạy command này, thì eksctl sẽ sử dụng AWS CloudFormation để tạo các infrastructure cần thiết và setup Master Node (Control Plane).
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/09.png">
- Trong quá trình chờ, Bạn có thể sử dụng giao diện quản lý AWS CloudFormation kiểm tra trạng thái.
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/03.png">
- Để kiểm tra, mình sẽ sử dụng kubectl:

```shell
kubectl get nodes
```

  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/10.png" alt="">

- Kiểm tra trên AWS Console để xem các Resource được tạo
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/11.png">
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/12.png">
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/13.png">

### Deploy mywebsite

- Tiếp theo, mình sẽ deploy Static Website vào EKS Cluster mình vừa tạo. Để deploy, đầu tiên mình sẽ chạy command sau:

```sh
kubectl create deployment mywebsite --image=minhtri2582/mywebsite
```
> Lưu ý: tên deployment **mywebsite**, chúng ra sẽ tên này trong phần 2

- Để có thể truy cập Website từ bên ngoài EKS cluster, chúng ta sẽ phải deploy một LoadBalancer Service vào cluster bằng command sau:

```sh
kubectl expose deployments/mywebsite --type=LoadBalancer --port=80
```

- Để xem thông tin về LoadBalancer trên, mình sẽ chạy command sau:

```sh
kubectl get svc
```

  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/14.png" alt="">

- Copy link trong External-ip vào trình duyệt sẽ truy cập được website:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/eks/15.png">

### Okay, vậy là mình đã deploy thành công Static Web Server vào EKS Cluster.

<br>

# 2 - Tạo CI/CD với AWS CodePipeline
Đây là sơ đồ triển khai CI/CD bằng công cụ AWS CodePipeline chúng ta sẽ làm trong phần này:
<img src="https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/cicd.png"/>

### 2.1 - Chuẩn bị (S3, IAM Roles, RBAC)
1. Tạo S3 bucket
   Cho CodePiple lưu artifact (tạo ra tập tin build.json sau mỗi lần build)
> **TODO:** Trí kiểm tra lại có thể bỏ S3 được không? build.json không dùng cho stage sau.
```
ACCOUNT_ID=$(aws sts get-caller-identity | jq -r '.Account')
aws s3 mb s3://eks-${ACCOUNT_ID}-codepipeline-artifacts
```

2. Tạo Role CodePipelineServiceRole

- CodePileline và CodeBuild cần phải có các IAM roles để tạo Docker, push image và tương tác với EKS cluster qua command kubectl.
  Tải các file json chính sách, sau đó tạo các role **eks-CodePipelineServiceRole, eks-CodePipelineServiceRole** và thêm inline policy từ terminal như sau:
```shell
wget https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/cpAssumeRolePolicyDocument.json

aws iam create-role --role-name eks-CodePipelineServiceRole --assume-role-policy-document file://cpAssumeRolePolicyDocument.json

wget https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/cpPolicyDocument.json

aws iam put-role-policy --role-name eks-CodePipelineServiceRole --policy-name codepipeline-access --policy-document file://cpPolicyDocument.json
```

3. Tạo Role CodeBuildServiceRole

```shell
wget https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/cbAssumeRolePolicyDocument.json

aws iam create-role --role-name eks-CodeBuildServiceRole --assume-role-policy-document file://cbAssumeRolePolicyDocument.json

wget https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/cbPolicyDocument.json

aws iam put-role-policy --role-name eks-CodeBuildServiceRole --policy-name codebuild-access --policy-document file://cbPolicyDocument.json
```
> **TODO:** Chụp hình show IAM roles đã được tạo trên AWS Console

4. Cho phép role eks-CodeBuildServiceRolequyền trong RBAC của EKS cluster
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
  metadata.creationTimestamp: XXXXXXXXX
  name: aws-auth
  namespace: kube-system
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


- Tập aws-auth.yaml sau khi thêm role <b>eks-CodeBuildServiceRole</b> có dạng như sau (**Lưu ý**: xóa dòng _metadata.creationTimestamp_)

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
> **TODO:** hướng dẫn fork source từ Github của Được để có quyền push code, tạo personal token

### 2.3 - Tạo CodePipeline bằng CloudFormation
Bước tiếp theo chúng ta sẽ tạo CodePipeline sử dụng công cụ AWS ClouFormation.
1. Download template CloudFormation tại: <br>
   https://raw.githubusercontent.com/minhtri2582/server-samples/3fd9f41672b171483db7ee495c834507991125ad/aws-journey/code_pipeline.yml

<br>

2. Trong màn hình quản trị CloudFormation - Click **New Task**

- Chọn file Upload: **code_pipeline.yml** đã download ở bước 1. Click **Next**:
  <img src="https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/CF-CreateTask.png"/>

<br>

- Nhập thông tin GitHub (đã tạo ở phần 2.2) và EKS Cluster (đã tạo ở phần 1):
  <img src="https://raw.githubusercontent.com/minhtri2582/server-samples/master/aws-journey/CF-Input.png"/>
  Click **Next** và **Create Task** để tiếp tục.

<br>

- Chờ khoảng 5 phút để CloudFormation tạo các resource _CodePipeLine_, _CodePipeLine_ và _ECR Repository_:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/CF-Progress.png"/>

  <br>

- Vào trang quản trị CodePipeline, chọn Pipelines. Bạn sẽ thấy Pipeline đầu tiên đang chạy:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/CP-List.png"/>

<br>

- Click vào tên pipeline (**eks-codepipeline-CodePipelineGithubXXXXXXX**) để xem chi tiết
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/CP-Details.png"/>

<br>

- Chờ đến khi Pipeline chạy thành công. Trong stage **Build** - Click **Details** để xem CodeBuild Job:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/aws-journey/CP-Success.png">

<br>

- Chúng ta có thể xem log output trong quá trình build và deploy:
  <img src="https://github.com/minhtri2582/server-samples/raw/master/CB-DetailSuccess.png"/>
> **TODO:** Chỉ rõ chỗ nào click để ra output này nha!

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

## Chúc mừng bạn đã hoàn thành bài lab!
