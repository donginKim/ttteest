# 통합 보안 모듈 배포 가이드



## 1. 배포 환경 구성

| 구분                           | 최소사양            | 권장사양              | 구성요소                                                     |
| ------------------------------ | :------------------ | :-------------------- | ------------------------------------------------------------ |
| 보안기술<br/> 컴포넌트<br/> VM | 2 vCPU<br/>8 GB RAM | 8 vCPU <br/>32 GB RAM | * 통합 연동 모듈 <br/>* 이미지 보안점검 도구 <br/>* 보안 프로파일 생성 도구<br/>* Docker 엔진<br/>   ㄴ kubernetes 노드의 Docker 엔진 버전과 동일<br/>* BOSH CLI<br/>* CF CLI <br/>* kubectl |
| DB VM                          | 1 vCPU<br/>2 GB RAM | 2 vCPU<br/>4 GB RAM   | * 통합 연동 모듈 Database                                    |



## 2. 설치 전 준비 사항

### 2.1. 선행 구성 요소

본 선행 구성 요소가 설치 되어 있어야 통합 보안 모듈을 사용 할 수 있다.

#### 2.1.1. 보안기술 컴포넌트 VM

##### Docker 엔진

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc && sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88

sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
apt-cache madison docker-ce

#  docker-ce | 5:18.09.1~3-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu  xenial/stable amd64 Packages
#  docker-ce | 5:18.09.0~3-0~ubuntu-xenial | https://download.docker.com/linux/ubuntu  xenial/stable amd64 Packages
#  docker-ce | 18.06.1~ce~3-0~ubuntu       | https://download.docker.com/linux/ubuntu  xenial/stable amd64 Packages
#  docker-ce | 18.06.0~ce~3-0~ubuntu       | https://download.docker.com/linux/ubuntu  xenial/stable amd64 Packages
#  ...
  
sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io
```

> <https://docs.docker.com/install/>

##### Docker-compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
```

> <https://docs.docker.com/compose/install/>

##### BOSH CLI

```bash
wget https://github.com/cloudfoundry/bosh-cli/releases/download/v5.5.0/bosh-cli-5.5.0-linux-amd64

chmod +x ./bosh*
sudo mv ./bosh* /usr/local/bin/bosh
```

> <https://bosh.io/docs/cli-v2-install/>

##### CF CLI 

```bash
wget -q -O - https://packages.cloudfoundry.org/debian/cli.cloudfoundry.org.key | sudo apt-key add -

echo "deb https://packages.cloudfoundry.org/debian stable main" | sudo tee /etc/apt/sources.list.d/cloudfoundry-cli.list

sudo apt-get update
sudo apt-get install cf-cli
```

>  <https://docs.cloudfoundry.org/cf-cli/install-go-cli.html>

##### kubectl

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

> <https://kubernetes.io/docs/tasks/tools/install-kubectl/>

#### 2.1.2. DB VM

##### Mysql DB

```bash
sudo apt-get update -y && sudo apt-get install mysql-server
```

> <https://dev.mysql.com/downloads/installer/>

#### 2.1.3. ETC

* Cloud Foundry
* Kubernetes
* Pipline
* Docker registry
* go v1.11.0 이상



## 3. 통합 연동 모듈 설치

### 3.1. BOSH UAA 사용자 등록

통합 연동 모듈이 사용 할 수 있는 사용자를 등록하여 루트파일시스템 조회, 등록을 할 수 있도록 등록해야 한다.

##### 등록방법

````bash
uaac target https://((UAA URL)):8443 --skip-ssl-validation
uaac token client get ((UAA ADMIN)) -s ((UAA ADMIN SECRET))
uaac client add ((CLIENT NAME)) --name ((CLIENT NAME)) -s ((CLIENT SECRET)) \
	--authorities "bosh.read, bosh.*.read, bosh.releases.upload"
	--authorized_grant_types "client_credentials"
	--scope "uaa.none"
````



### 3.2. Cloud Foundry 사용자 등록

통합 연동 모듈이 사용 할 수 있는 사용자를 등록하여 앱 배포, 조회, 삭제를 할 수 있도록 등록해야 한다.

##### 등록 방법

````bash
cf login -a https://api.((DOMAIN)) -u ((ADMIN USER)) -p ((ADMIN PASSWORD))
cf target -o ((CF ORG)) -s ((CF SPACE))

cf create-user ((USER NAME)) ((USER PASSWORD))
cf set-org-role ((USER NAME)) ((CF ORG)) OrgAuditor
cf set-space-role ((USER NAME)) ((CF ORG)) ((CF SPACE)) SpaceDeveloper
````



### 3.3. Credhub 등록

통합 연동 모듈에서 사용하는 사용자 인증을 위하여 등록해야 한다.

##### 등록방법

```bash
credhub set --type password --name ((CREDHUB NAME)) --password ((USER PASSWORD[CF USER PASSWORD]))
```



### 3.4. kubernetes 사용자 등록

통합 연동 모듈에서 사용 할 수 있는 사용자를 등록하여 pod 배포, 조회, 삭제를 할 수 있도록 등록해야 한다.

##### 등록방법

```bash
kubectl create serviceaccount ((USER NAME))
kubectl create clusterrole ((USER NAME)) --verb=get --verb=list --verb=watch
kubectl create clusterrolebinding ((USER NAME)) \
  --serviceaccount=((USER NAME)) \
  --clusterrole=((USER NAME))

TOKEN=$(kubectl describe secrets "$(kubectl describe serviceaccount ((USER NAME)) | grep -i Tokens | awk '{print $2}')" | grep token: | awk '{print $2}')

kubectl config set-credentials ((CONFIG NAME)) --token=$TOKEN
kubectl config set-context podreader --cluster=$(kubectl config current-context) --user=((CONFIG NAME))
kubectl config use-context ((CONFIG NAME))
```



### 3.5. Database 생성

통합 연동 모듈에서 사용하는 데이터베이스를 생성해야 한다.

##### 생성방법

````bash
mysql > create database paas_integrated_interface;
````



### 3.6. 통합 연동 모듈 config 설정

통합 연동 모듈과 연동할 설정을 config/config.json에 설정해야 한다.

##### 설정방법

````json
{
    "Policy" : {
        "Docker": "((DETECTION LEVEL))",
        "RootFS": "((DETECTION LEVEL))",
        "Droplet": "((DETECTION LEVEL))"
    },
    "Database": {
        "Name": "paas_integrated_interface",
        "Host": "((DATABASE HOST))",
        "Port": "3306",
        "Username": "((DATABASE USER NAME))",
        "Password": "((DATABASE USER PASSWORD))"
    },
    "ApiServer": {
        "Port": ":1323",
        "TempDir": "/tmp/integrated_interface_module",
        "Mail": {
            "Username": "((USER MAIL))",
            "Password": "((USER PASSWORD))",
            "Host": "smtp.gmail.com",
            "Port": "587",
            "sysAdmin": "((SYSTEM ADMIN MAIL))"
        },
        "Slack": {
            "Host": "https://slack.com",
            "Token": "xoxp-517708431154-516972874240-605426914178-6063729555e83da713c5aed15942f913",
            "ChannelName": "inspector_alarms",
            "ChannelListApi": "/api/channels.list",
            "PostMessageApi": "/api/chat.postMessage"
        },
        "Scheduler": {
            "IntervalHour": 2
        }
    },
    "Cf": {
        "Api": "https://api.((CF DOMAIN))",
        "Username": "((CF USER NAME))",
        "Password": "((CF USER PASSWORD))",
        "CredHubName": "((CRED HUB NAME))",
        "DeploymentName": "((BOSH DEPLOYMENT NAME))"
    },
    "Bosh": {
        "BoshAddress": "https://((BOSH URL)):25555",
        "Client": "((BOSH CLIENT))",
        "ClientSecret": "((BOSH CLIENT SECRET))",
        "Environment": "((BOSH ENVIRONMENT ALIAS))"
    },
    "Docker": {
        "Api": "((DOCKER API ADDRESS))",
        "Username": "((DOCKER USER NAME))",
        "Password": "((DOCKER USER PASSWORD))",
        "Email" : "((DOCKER USER EMAIL))",
        "ImageName" : "((COPY PROFILE IMAGE NAME))"
    },
    "Kubernetes": {
        "Api": "((KUBERNETES API ADDRESS))",
        "Username": "((KUBERNETES USER NAME))",
        "Password": "((KUBERNETES USER PASSWORD))",
        "NameSpace": "((KUBERNETES NAMESPACE NAME))",
        "ProfilePath" : "workspace/nsr/profile",
        "YamlPath" : "workspace/nsr/yaml"
    },
    "Security": {
        "Api": "http://((CLAIR URL)):6060",
        "Path": "workspace/nsr/nsrclair_install",
        "Config": "config.yaml",
        "ResultPath": "workspace/nsr/nsrclair_results",
        "ImgRegPath": "image-reg"
    },
    "Profile": {
        "Path": "workspace/nsr/profGen",
        "ResultPath": "workspace/nsr/yaml",
        "ProfileContainers": "rule-set",
        "ApplicationSeccomp" : "container.seccomp.security.alpha.kubernetes.io",
        "SeccompPath" : "localhost"
    }
}
````



### 3.7. 통합 연동 모듈 설치

#### 3.7.1. 통합 연동 모듈 다운로드

```bash
mkdir -p workspace/nsr/src && cd workspace/nsr/src
git clone https://github.com/CrossentCloud/paas-integrated-interface.git
```

#### 3.7.2. GOPATH 설정

```bash
cd ~/workspace/nsr

export GOPATH=$PWD
export PATH=$GOPATH/bin:$PATH:
```

#### 3.7.3. 보안 점검 도구, 프로파일링 도구 위치

```bash
#보안 점검 도구
cd ~/workspace/nsr/nsrclair_install

#프로파일링 도구
cd ~/workspace/nsr/profGen
```

#### 3.7.4. 통합 연동 모듈 실행

```bash
cd ~/workspace/nsr/src/paas-integrated-interface
go run cmd/server.go
```

