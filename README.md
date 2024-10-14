# Jenkins와 AWS RDS, S3를 이용한 웹 서비스 CI/CD 🔁

---
<br>

## 👨‍👨‍👧 개발팀원  

| <img src="https://avatars.githubusercontent.com/u/65991884?v=4" width="80"> | <img src="https://avatars.githubusercontent.com/u/90691610?v=4" width="80"> | <img src="https://avatars.githubusercontent.com/u/79312705?v=4" width="80"> |
|:---:|:---:|:---:|
| [류채현](https://github.com/RyuChaeHyun) | [이석철](https://github.com/SeokCheol-Lee) | [김상민](https://github.com/isshomin) |

---

<br>

## 개요 ✒

Jenkins, GitHub, 및 AWS S3를 활용하여 CI/CD 프로세스를 자동화하고, 빌드된 .jar 파일을 S3에 **자동으로 업로드하여 배포 및 관리 효율성을 극대화**합니다. 또한, AWS RDS와 연동하여 **데이터베이스 관리 성능을 향상**시킵니다.

---

<br>

## 실습환경 🏞

```ruby
virtualbox: version 7.0
ubuntu: version 22.04.4 LTS
docker: version 27.3.1
minikube: version v1.34.0
Jenkins: version 2.462.3
Amazon RDS, Amazon S3
```

---

<br>

## Jenkins CI/CD 환경구축 🍵

#### 1️⃣) docker compose file을 이용해 Jenkins를 설치해줍니다.

```yaml
#compose.yaml
services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    user: root
    container_name: myjenkins
    ports:
      - 8080:8080
      - 50001:50000
    volumes:
      - ./jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - TZ=Asia/Seoul
```

<br>

#### 2️⃣) Jenkins 포트포워딩을 해줍니다. 

![image](https://github.com/user-attachments/assets/a1a00939-be36-4e0a-94de-60a9988e1fe5)


<br>

#### 3️⃣) 필요한 권한만 부여하고, 실수나 악의적인 접근을 방지하기 위해 root 계정 이외에 사용자 계정을 생성해줍니다.

```mysql
use fisa;

-- 로컬 user 생성
CREATE USER 'user01'@'localhost' identified BY 'User_Password';

-- 외부 접속 user 생성
CREATE USER 'user01'@'%' identified BY 'User_Password';

-- user 목록 확인
use mysql;
select user, host from user;


-- fisa 데이터베이스에 권한 부여
GRANT ALL PRIVILEGES ON fisa.* TO 'user01'@'localhost';
FLUSH PRIVILEGES;

-- user 사용자 권한 조회
show grants  for 'user01'@'localhost';
```

![image](https://github.com/user-attachments/assets/ba559153-5c61-42ec-a227-b60940fd606b)

<br>

#### 4️⃣) 소스파일의 application.properties에서 기밀정보를 암호화 해줍니다.

```java
spring.datasource.url=${DB_URL}
spring.datasource.username=${DB_USERNAME}
spring.datasource.password=${DB_PASSWORD}
```

<br> 

#### 5️⃣) .bashrc 파일에 암호화된 DB정보를 매핑해줍니다.

```ruby
sudo nano .bashrc
```

###### ~/.bashrc
```ruby
# export 추가
export DB_URL="jdbc:mysql://<RDS_EndPoint>:3306/fisa?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true"
export DB_USERNAME=DB_ID
export DB_PASSWORD=DB_Password
```

<br>

#### 6️⃣) ngrok을 활용하여 github repository에 hook 해줍니다. 


[참고](https://github.com/isshomin/jenkins_auto_deploy)

<br>

#### 7️⃣) Jenkins에 새 item을 생성하고 파이프라인 스크립트를 작성해줍니다.
 

```ruby
pipeline {
    agent any

    stages {
        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/isshomin/RDS_jenkins_test.git'
            }
        }
          
        stage('Build') {
            steps {
                sh 'chmod +x gradlew'                    
                sh './gradlew clean build -x test'
            }
        }
        
        stage('Copy jar') { 
            steps {
                script {
                    def jarFile = 'build/libs/step18_empApp-0.0.1-SNAPSHOT.jar'
                    def destDir = '/var/jenkins_home/'                   
                    sh "cp ${jarFile} ${destDir}"
                }
            }
        }
		
		stage('Upload to S3') {
            steps {
                script {
                    def jarFile = 'build/libs/step18_empApp-0.0.1-SNAPSHOT.jar'
                    def bucketName = 'ce04-bucket-01'
                    def s3Path = 'jar' // S3 내 경로

                    // AWS CLI로 JAR 파일을 S3로 업로드
                    sh "aws s3 cp ${jarFile} s3://${bucketName}/${s3Path}/"
                }
            }
        }
    }
}
```

---

<br>

## AWS 환경구축 ☁

#### 1️⃣) RDS의 엔드포인트를 이용하여 DBeaver에 연동해줍니다.

![image](https://github.com/user-attachments/assets/b64f57b5-b6c4-459b-b037-b522004f499d)


<br>

#### 2️⃣) docker 내부의 Jenkins 컨테이너에 aws cli를 설치해줍니다.

```ruby
#Jenkins 컨테이너 내부 Bash shell 사용
docker exec -it -u root myjenkins bash

# aws-cli 설치
apt update
apt install awscli

#aws-cli 설치확인
aws --version
```

![image](https://github.com/user-attachments/assets/0539db62-c179-48e1-afed-98e9cc08bb13)

<br>

#### 3️⃣) aws configure를 명령어를 통해 AWS 계정과 관련된 기본 설정을 해줍니다.

```ruby
aws configure

AWS Access Key ID [None]: <your-access-key-id>
AWS Secret Access Key [None]: <your-secret-access-key>
Default region name [None]: ap-northeast-2
Default output format [None]: json
```

<br>

#### 4️⃣) 계정 설정이 제대로 되었는지 확인합니다.

```ruby
aws s3 ls | grep ce04
```

![image](https://github.com/user-attachments/assets/44a4bd33-2d79-4ec6-b6ba-a435ba4673d9)

---

<br>

## 실행 확인 ➡

#### 1️⃣) hook 되어 있는 repository의 소스파일을 수정하여 Jenkins를 통해 제대로 CI/CD가 되는지 확인합니다.

![image](https://github.com/user-attachments/assets/720d525e-640b-45b8-ab4e-8d71c5cc37ef)

<br>

#### 2️⃣) AWS S3에 접속하여 jar 폴더에 제대로 업로드 되었는지 확인합니다.

![image](https://github.com/user-attachments/assets/d15e068d-e744-4c50-a713-aba057fd4aea)

<br>

---

<br>

## 요약 📩

Jenkins와 GitHub을 연동하여 CI/CD 파이프라인을 구축하고, **코드가 변경될 시 자동 빌드 및 테스트를 실행**합니다. 빌드된 .jar 파일은 AWS S3에 자동으로 업로드되며, AWS RDS와 연동하여 데이터베이스를 관리합니다. 결과적으로 **배포와 관리 효율성을 극대화하고 자동화된 워크플로우를 구현**했습니다.

<br>

---

<br>

## 트러블슈팅 🎯🔫

### 1️⃣) Jenkins 파이프라인 스크립트를 빌드 시 무기한 대기에 빠진 현상 ⛈

![image](https://github.com/user-attachments/assets/694d9fbe-c55f-4663-aa03-2f33ece876e3)


<br>

### 원인추론 ⛅

#### 잘못된 환경 변수로 인해 경로 문제, 인증 실패, 또는 필수 파일 누락 등이 발생할 수 있으며, 이로 인해 스크립트가 정상적으로 실행되지 않고 무기한 대기 상태에 빠질 수 있다는 것을 알았습니다.

[출처](https://stackoverflow.com/questions/58848793/stuck-in-sh-stage-and-then-process-apparently-never-started-in-home-jenkins-j)

<br>

### 해결 🌞

#### 기존의 aws-cli 환경 변수를 삭제하고 Jenkins 컨테이너 내부에 aws-cli를 설치하여 해결하였습니다.

![image](https://github.com/user-attachments/assets/e44ea415-0f35-45b7-8940-856e1601f9b1)

---
