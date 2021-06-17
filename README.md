# CI/CD với gitlab

## I. Trước khi đọc bài viết này, hãy đảm bảo có đủ các yếu tố sau

- Một lập trình viên A
- Một project B
- Một gitlab server C (cụ thể ở đây là [git] này)
- Một máy chủ D (dùng để deploy, chạy ứng dụng)
- Một máy chủ E dùng để làm gitlab runner (D và E có thể là một)

## II. Tạo ra một gitlab runner E
Gitlab runner là gì?

> Một Runner có thể là một máy ảo (VM), một VPS, một bare-metal,
> một docker container hay thậm chí là một cluster container
> Runners giao tiếp với nhau thông qua API,
> vì vậy yêu cầu duy nhất là máy chạy Runner có quyền truy cập Gitlab server.

Trong hướng dẫn này sẽ dùng docker để chạy một runner

##### 1.1 Tạo file docker-compose.yml với nội dung

docker-compose.yml
```yaml
version: '3.3'
services:
  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    restart: unless-stopped
    container_name: gitlab-runner-c
    environment:
      TZ: UTC
```

##### 1.2 Chạy lệnh để thực thi docker-compose.yml

Thư mục chạy lệnh là thư mục chưa file docker-compose.yml
```
docker-compose up -d
```

![Product Name Screen Shot][docker-compose-up-d]

##### 1.3 Double-check gitlab-runner đã chạy hay chưa

Kiểm tra các container đang chạy
```
docker ps
```

![Product Name Screen Shot][docker-ps]

##### 1.4 SSH vào gitlab-runner vừa tạo

Đối với gitlab-runner chạy bằng máy vật lý hoặc máy ảo có địa chỉ IP XXX.YYY.ZZZ.WWW
```
ssh user@XXX.YYY.ZZZ.WWW
```

Đối với gitlab-runner trong hướng dẫn này
```
docker exec -it gitlab-runner-c bash
```

##### 1.5 Đăng kí máy gitlab-runner với server gitlab
Sau khi đã ssh vào gitlab-runner
```
gitlab-runner register
```

<ins>Enter the GitLab instance URL</ins>: Địa chỉ IP của server gitlab (Ở đây sẽ dùng luôn cây nhà lá vườn [git] )

<ins>Enter the registration token</ins>: Mỗi một project (hoặc group project) sẽ có token riêng. Vào trang chính của project trên gitlab server (http://gitlab.com/project-name) -> Settings -> CI/CD -> Runners -> Expand -> Use the following registration token during setup

<ins>Enter a description for the runner</ins>: Mô tả chung chung (Ví dụ: runner for demo)

<ins>Enter tags for the runner (comma-separated)</ins>: Tag dành cho runner, dùng để chỉ định nhánh nào, step nào, pipeline nào sử dụng runner với tag này (Ví dụ: production-runner, runner-carticket) ở đây sẽ dùng <b>demo-runner</b>

<ins>Enter an executor: shell, ssh, docker-ssh+machine, custom, docker, docker-ssh, parallels, virtualbox, docker+machine, kubernetes</ins>: ựa chọn một executor ở demo này sẽ sử dụng docker, executor là một trình thực thi ngữ cảnh. Một gitlab runner có thể chứa nhiều executor https://docs.gitlab.com/runner/executors/ 

<ins>Enter the default Docker image (for example, ruby:2.6)</ins>: Mặc định cần có một loại image cho executor (Chỉ có tùy chọn này nếu executor là docker). Ở demo này sẽ sử dụng <b>docker:20.10.7-dind</b>

Kết quả 

![Product Name Screen Shot][gitlab-runner-registry]

## Hiểu một cách đơn giản: Khi A sử dụng file deploy (B.jar hoặc B.war) để deloy lên D có bao nhiêu bước thì E sẽ làm bấy nhiêu bước (hoặc nhiều hơn)

Nhìn bảng dưới để có cái nhìn tổng quan:

| STT | A | E |
|---| ------ | ------ |
|1| Checkout nhánh cần deploy | git checkout master |
|2| Lấy code mới nhất về | git pull origin |
|3| Build file deploy | mvn clean package |
|4| Đẩy file build lên | scp B.jar root@C:/tmp |
|5| Logout vào C | ssh root@C |
|6| Copy file build vào thư mục deploy | cp /tmp/B.jar <path>/B.jar |
|7| Chạy câu lệnh deploy | java -jar B.jar |
|8| Logout khỏi C | exit |

E sẽ không tự động thực hiện các công việc trên nếu không có chỉ thị từ file gitlab-ci.yaml.

gitlab-ci.yaml là file hướng dẫn cho E biết, vào thời điểm nào thì làm việc gì

##  IV. Cấu hình gitlab-ci.yaml

Chúng ta sẽ dùng file gitlab-ci.yaml ở dưới đây để phân tích từng block.

```yaml
variables:
  MAVEN_OPTS: '-Djava.awt.headless=true -Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository'
  MAVEN_CLI_OPTS: '--batch-mode --errors --fail-at-end --show-version'
  PACKAGE_NAME: 'demo-eros'
  BUILD_PATH: 'build'
  PASSWORD: 123

cache:
  paths:
    - ./.m2/repository

stages:
  - test
  - build
  - deploy

test:
  image: maven:3.6-openjdk-11-slim
  stage: test
  before_script:
    - mkdir -p $CI_PROJECT_DIR/.m2/repository
  script:
    - 'mvn -s maven/settings.xml --batch-mode -U clean verify $MAVEN_CLI_OPTS'
  tags:
      - demo-runner
build:
  image: maven:3.6-openjdk-11-slim
  stage: build
  before_script:
    - mkdir -p $CI_PROJECT_DIR/.m2/repository
  script:
    - 'mvn -s maven/settings.xml --batch-mode -U clean package -DskipTests'
    - 'mkdir $BUILD_PATH'
    - 'cp eros-consumer/target/eros-consumer*.jar $BUILD_PATH/$PACKAGE_NAME.jar'
  artifacts:
    expire_in: 8h
    paths:
      - '$BUILD_PATH/$PACKAGE_NAME.jar'
  tags:
      - demo-runner
deploy:
  image: ictu/sshpass:latest
  stage: deploy
  script:
    - 'sshpass -p "$PASSWORD" scp -o StrictHostKeyChecking=no $BUILD_PATH/$PACKAGE_NAME.jar root@xx.xx.xx.xx:"/tmp/"'
    - 'sshpass -p "$PASSWORD" ssh -o StrictHostKeyChecking=no -tt root@xx.xx.xx.xx "bash -s" < ./deploy/deploy.sh'
  tags:
      - demo-runner
```

<b>variables</b>:  giống như trong java hay bất kì ngôn ngữ lập trình nào khác, variable khai các biến cùng giá trị của nó

<b>cache</b>: trong ví dụ này, ứng dụng java sẽ sử dụng maven để build ra file deploy. Các thư viện của maven sẽ được lưu trong paths của cache (cũng có thể áp dụng với node_module của các ứng dụng react)

<b>stages</b>: liệt kê pipeline có bao nhiêu bước. Trong ví dụ này là 3: test, build, deploy. Các stages này được khai báo tuy nhiên triển khai sẽ được viết độc lập thành những block ở dưới. Số lượng stages cũng được hiển thị trên pipeline của gitlab-server
![stages][stages]

<b>test</b>: test block sẽ khác với những block variables, cache hay stages. Nó không phải định nghĩa của gitlab, mà người dùng định nghĩa. Trong stages chúng ta có khai báo - test thì ở đây sẽ là triển khai của nó. Quay lại ví dụ, một ứng dụng java cần build bằng maven, tuy nhiên khi khởi tạo gitlab-runner, chúng ta đã khai báo:

```
Enter the default Docker image (for example, ruby:2.6):
docker:20.10.7-dind
```

Tuy nhiên, image <b>docker:20.10.7-dind</b> không có sẵn maven nên thay vào đó, trong test block chúng ta cần phải sử dụng một executor có maven

```yaml
image: maven:3.6-openjdk-11-slim
```

Tiếp theo:

```yaml
stage: test
```
-> test block là tên bất kì, có thể là test-script, test-step. Còn stage: test có nghĩa block này là triển khai có test đã khai báo ở <b>stages</b>

```yaml
before_script:
  - mkdir -p $CI_PROJECT_DIR/.m2/repository
```
->  Ghi đè một tập hợp các lệnh được thực hiện trước công việc. Trong ví dụ này là tạo ra thư mục để lưu các thư viện maven

```yaml
script:
  - 'mvn -s maven/settings.xml --batch-mode -U clean verify $MAVEN_CLI_OPTS'
```
-> script chính chạy sau khi before_script thực thi thành công. $MAVEN_CLI_OPTS là biến đã khai báo tại <b>variables</b>

```yaml
tags:
    - demo-runner
```
-> pipeline này chỉ cho phép các runner trong tags chạy. demo-runner đã được khai báo khi khởi tạo gitlab-runner

<b>build</b>: trong stages build cũng không khác nhiều so với test, điểm khác ở đây là
```yaml
artifacts:
  expire_in: 8h
  paths:
    - '$BUILD_PATH/$PACKAGE_NAME.jar'
```
-> Sau khi stages build hoàn tất, sẽ upload file artifacts lên gitlab-server và có thể sử dụng trong 8h. Cũng có thể download trên gitlab file build này
![artifacts][artifacts]

<b>deploy</b>: stages cuối này việc cũng rất đơn giản là scp file build lên server D và chạy script để build

<b>Việc sử dụng scp và deploy bằng file sh (trong ví dụ này) là một chiến lược đơn giản nhưng không phù hợp để áp dụng nếu gitlab runner và server production là một</b>

[git]: <https://gitlab.com/>
[docker-compose-up-d]: docker-compose-up-d.png
[docker-ps]: docker-ps.png
[gitlab-runner-registry]: gitlab-runner-registry.png
[stages]: stages.png
[artifacts]: artifacts.png
