![Alt text](/chan/image/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4.png)

Chapter 3
=========
 컨테이너를 다루는 표준 아키텍처, 쿠버네티스
------------------------------------------

---

# 0. Intro
**컨테이너 인프라 환경** 

> 리눅스 운영 체제의 커널 하나에서 여러 개의 컨테이너가 격리된 상태로 실행되는 인프라 환경을 말합니다. 
> 
> **컨테이너**
>> 하나 이상의 목적을 위해 독립적으로 작동하는 프로세스입니다.

---

# 1. 쿠버네티스 이해하기
**쿠버네티스**
> 컨테이너 오케스트레이션을 위한 솔루션
>> **오케스트레이션**
>>> 복잡한 단계를 관리하고 요소들의 유기적인 관계를 미리 정의해 손쉽게 사용하도록 서비스를 제공하는 것을 의미합니다.

### 1). Vagrantfile
> 베이그런트 프로비저닝을 위한 정보를 담고 있는 메인 파일입니다.

```ruby
Vagrant.configure("2") do |config| # "2"는 베이그런트에서 루비로 코드를 읽어 들여 실행할 때 작동하는 API 버전이고, 뒤의 do |config|는 베이그런트 설정의 시작을 알립니다.
  N = 3 # 작업을 수행할 워커 노드의 수
  Ver = '1.18.4' # 설치할 쿠버네티스 버젼

  #=============#
  # Master Node #
  #=============#

    config.vm.define "m-k8s" do |cfg| # 버추어박스에서 보이는 가상 머신을 "m-k8s"로 정의하고, do |cfg|를 추가해 원하는 설정으로 변경합니다. 
    # 이렇게 do|이름|으로 시작한 작업은 end로 종료합니다.
      cfg.vm.box = "sysnet4admin/CentOS-k8s" # 기본값 config.vm.box를 do |cfg|에 적용한 내용을 받아 cfg.vm.box로 변경합니다.
      cfg.vm.provider "virtualbox" do |vb| 
      # 베이그런트의 프로바이더가 버추얼박스라는 것을 정의합니다. 
      # 프로바이더는 베이그런트를 통해 제공되는 코드가 실제로 가상 머신으로 배포되게 하는 스프트웨어입니다.
      # 버추얼박스가 여기에 해당합니다. 다음으로 버추얼박스에서 필요한 설정을 정의하는데, 그 시작을 do |vb|로 선언합니다.
        vb.name = "m-k8s(github_SysNet4Admin)"
        vb.cpus = 2
        vb.memory = 3072
        vb.customize ["modifyvm", :id, "--groups", "/k8s-SgMST-1.13.1(github_SysNet4Admin)"]
      end # 버추얼박스에 생성한 가상 머신의 이름, CPU 수, 메모리 크기, 소속된 그룹을 명시합니다.
      # 그리고 end를 적어 버추얼박스 설정이 끝났음을 알립니다.
      cfg.vm.host_name = "m-k8s" # 여기서부터는 가상 머신 자체에 대한 설정으로, do |cfg|에 속한 작업입니다.
      # 호스트의 이름(m-k8s)을 설정합니다.
      cfg.vm.network "private_network", ip: "192.168.1.10" 
      # 호스트 전용 네트워크를 private_network로 설정해 eth1 인터페이스를 호스트 전용으로 구성하고 IP 지정합니다.
      cfg.vm.network "forwarded_port", guest: 22, host: 60010, auto_correct: true, id: "ssh"
      # ssh 통신은 호스트 60010번을 게스트 22번으로 전달되도록 구성합니다. true는 중복 대비
      cfg.vm.synced_folder "../data", "/vagrant", disabled: true 
      cfg.vm.provision "shell", path: "config.sh", args: N # config.sh를 게스트에서 실행합니다.
      cfg.vm.provision "shell", path: "install_pkg.sh", args: [ Ver, "Main" ]
      # args: [Ver, "Main"] 코드를 추가해 쿠버네티스 버전 정보와 main이라는 문자를 install_pkg.sh로 넘깁니다.
      # Ver 변수는 각 노드에 해당 버전의 쿠버네티스 버전을 설치하게 합니다.
      # 두 번째 인자인 Main 문자는 install_pkg.sh에서 조건문으로 처리해 마스터 노드에만 이 책의 전체 실행 코드를 내려받게 합니다.
      cfg.vm.provision "shell", path: "master_node.sh"
    end

  #==============#
  # Worker Nodes #
  #==============#

   # 추가한 N대 CentOS에 대한 구성입니다. 
  (1..N).each do |i| # 1부터 N까지 N개의 인자를 반복해 i로 입력
    config.vm.define "w#{i}-k8s" do |cfg| # {i} 값이 치환됨
      cfg.vm.box = "sysnet4admin/CentOS-k8s" 
      cfg.vm.provider "virtualbox" do |vb|
        vb.name = "w#{i}-k8s(github_SysNet4Admin)" # {i} 값이 치환됨
        vb.cpus = 1
        vb.memory = 2560 
        vb.customize ["modifyvm", :id, "--groups", "/k8s-SgMST-1.13.1(github_SysNet4Admin)"]
      end
      cfg.vm.host_name = "w#{i}-k8s"
      cfg.vm.network "private_network", ip: "192.168.1.10#{i}"
      cfg.vm.network "forwarded_port", guest: 22, host: "6010#{i}", auto_correct: true, id: "ssh"
      cfg.vm.synced_folder "../data", "/vagrant", disabled: true
      cfg.vm.provision "shell", path: "config.sh", args: N
      cfg.vm.provision "shell", path: "install_pkg.sh", args: Ver
      cfg.vm.provision "shell", path: "work_nodes.sh"
    end
  end

end
```

### 2). config.sh
> kubeadm으로 쿠버네티스를 설치하기 위한 사전 조건을 설정하는 스크립트 파일입니다. 쿠버네티스의 노드가 되는 가상 머신에 어떤 값을 설정하는지 알아보겠습니다.

```ruby
#!/usr/bin/env bash

# vim configuration 
echo 'alias vi=vim' >> /etc/profile

# swapoff -a 쿠버네티스의 설치 요구 조건을 맞추기 위해 스왑되지 않도록 설정
swapoff -a
# 시스템이 다시 시작되더라도 스왑되지 않도록 설정합니다.
sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab

# kubernetes repo
gg_pkg="packages.cloud.google.com/yum/doc" # 쿠버네티스의 리포지터리를 설정하기 위한 경로가 너무 길어지지 않게 경로를 변수로 처리합니다.
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://${gg_pkg}/yum-key.gpg https://${gg_pkg}/rpm-package-key.gpg
EOF # 쿠버네티스를 내려받을 리포지터리를 설정하는 구문입니다.

# selinux가 제한적으로 사용되지 않도록 permissive 모드로 변경합니다.
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 브리지 네트워크를 통화하는 IPv4와 IPv6의 패킷을 iptables가 관리하게 설정합니다.
# 파드(Pod, 쿠버네티스에서 실행되는 객체의 최소 단위)의 통신을 iptables로 제어합니다.
# 필요에 따라 IPVS 같은 방식으로도 구성할 수도 있습니다.
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF 
modprobe br_netfilter
# br_netfilter 커널 모듈을 사용해 브리지로 네트워크를 구성합니다.
# 이때 IP 마스커레이드를 사용해 내부 네트워크와 외부 네트워트를 분리합니다.
# IP 마스커레이드는 쉽게 설명하면 커널에서 제공 NAT(Network Addresas Translation) 기능으로 이해.
# 실제로는 br_netfilter를 적용함으로써 위에서 적용한 iptables가 활성화

# 쿠버네티스 안에서 노드 간 통신을 이름으로 할 수 있도록 각 노드의 호스트이름과 IP를 /etc/hosts에 설정합니다.
# 이때 워커 노드는 Vagrantfile에서 넘겨받은 N변수로 전달된 노드 수에 맞게 동적으로 생성합니다.
echo "192.168.1.10 m-k8s" >> /etc/hosts
for (( i=1; i<=$1; i++  )); do echo "192.168.1.10$i w$i-k8s" >> /etc/hosts; done

# config DNS 외부와 통신할 수 있게 DNS 서버를 지정합니다.
cat <<EOF > /etc/resolv.conf
nameserver 1.1.1.1 #cloudflare DNS
nameserver 8.8.8.8 #Google DNS
EOF
```

### 3). install_pkg.sh
> 클러스터를 구성하기 위해서 가상 머신에 설치돼야 하는 의존성 패키지를 명시.
> 또한 실습에 필요한 소스 코드를 특정 가상 머신(m-k8s) 내부에 내려받도록 설정돼 있습니다.

```ruby
#!/usr/bin/env bash

# install packages 
yum install epel-release -y
yum install vim-enhanced -y
yum install git -y # 깃허브에서 코드를 내려받을 수 있게 깃을 설치합니다.

# 도커 설치
yum install docker -y && systemctl enable --now docker

# 쿠버네티스를 구성하기 위해 클러스터 설치
yum install kubectl-$1 kubelet-$1 kubeadm-$1 -y
systemctl enable --now kubelet

# git clone _Book_k8sInfra.git 
if [ $2 = 'Main' ]; then
  git clone https://github.com/sysnet4admin/_Book_k8sInfra.git
  mv /home/vagrant/_Book_k8sInfra $HOME
  find $HOME/_Book_k8sInfra/ -regex ".*\.\(sh\)" -exec chmod 700 {} \;
fi
```

### 4). master_node.sh
> 1개의 가상 머신(m-k8s)을 쿠버네티스 마스터 노드로 구성하는 스크립트입니다.
> 여기서 쿠버네티스 클러스터를 구성할 때 꼭 선택해야 하는 컨테이너 네트워크 인터페이스(CNI)도 함께 구성합니다.

```ruby
#!/usr/bin/env bash

# kubeadm을 통해 쿠버네티스의 워커 노드를 받아들일 준비를 합니다.
# 먼저 토큰을 123456.1234567890123456으로 지정하고 ttl(time to live)을 0으로 설정해서 기본값인 24시간 후에 토큰이 지속되게 유지합니다.
# 그리고 워커노드가 정해진 토큰으로 들어오게 합니다.
# 쿠버네티스가 자동으로 컨테이너에 부여하는 네트워크를 172.16.0.0/16으로 제공하고, 워커 노드가 접속하는 API 서버의 IP를 192.168.1.10으로 지정해 워커 노드들이 자동으로 API 서버에 연결되게 합니다.
kubeadm init --token 123456.1234567890123456 --token-ttl 0 \
--pod-network-cidr=172.16.0.0/16 --apiserver-advertise-address=192.168.1.10 

# 마스터 노드에서 현재 사용자가 쿠버네티스를 정상적으로 구동할 수 있게 설정 파일을 루트의 홈디렉터리(/root)에 복사하고 쿠버네티스 사용자에게 권한을 줍니다.
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 컨테이너 네트워크 인터페이스인 캘리코의 설정을 적용해 쿠버네티스의 네트워크를 구성합니다.
kubectl apply -f \
https://raw.githubusercontent.com/sysnet4admin/IaC/master/manifests/172.16_net_calico.yaml
```

### 5). worker_node.sh
> 1개의 가상 머신(m-k8s)을 쿠버네티스 마스터 노드로 구성하는 스크립트입니다.
> 여기서 쿠버네티스 클러스터를 구성할 때 꼭 선택해야 하는 컨테이너 네트워크 인터페이스(CNI)도 함께 구성합니다.

```ruby
#!/usr/bin/env bash

# kubeadm을 이용해 쿠버네티스 마스터 노드에 접속합니다.
# 이때 연결에 필요한 토큰을 기존에 마스터 노드에서 생성한 123456.1234567890123456을 사용합니다.
# 간단하게 구성하기 위해 --discovery-token-unsafe-skip-ca-verification으로 인증을 무시하고,
# API 서버 주소인 192.168.1.10으로 기본 포트 번호인 6443번 포트에 접속하도록 설정합니다.
kubeadm join --token 123456.1234567890123456 \
             --discovery-token-unsafe-skip-ca-verification 192.168.1.10:6443
```

---

# 2. 관리자나 개발자가 파드를 배포할 때

## Master Node

### 0). kubectl
> 쿠버네티스 클러스터에 명령을 내리는 역할
> 다른 구성 요소들과 다르게 바로 실행되는 명령 형태인 바이너리로 배포되기 때문에 마스터 노드에 있을 필요는 없습니다.

### 1). API 서버
> 쿠버네티스 클러스터의 중심 역할을 하는 통로입니다.
> 주로 상태 값을 저장하는 etcd와 통신하지만, 그 밖의 요소들 또한 API 서버를 중심에 두고 통신하므로 API 서버의 역할이 매우 중요합니다.
> 회사에 비유하면 모든 직원과 상황을 관리하고 목표를 설정하는 관리자에 해당합니다.

### 2). etcd
> 구성 요소들의 상태 값이 모두 저장되는 곳.
> 회사의 관리자가 모든 보고 내용을 기록하는 노트라고 생각하면 됨.
> 실제로 etcd 외의 다른 구성 요소는 상태 값을 관리하지 않습니다.
> etcd의 정보만 백업돼 있다면 긴급한 장애 상황에서도 쿠버네티스 클러스터는 복구할 수 있습니다.
> 또한 etcd는 분산 저장이 가능한 key-value 저장소이므로, 복제해 여러 곳에 저장해 두면 하나의 etcd에서 장애가 나더라도 시스템의 가용성을 확보할 수 있습니다.

### 3). 컨트롤러 매니저
> 쿠버네티스 클러스터의 오브젝트 상태를 관리.
> 통신 문제, 상태 체크와 복구

### 4). 스케쥴러
> 노드의 상태와 자원, 레이블, 요구 조건 등을 고려해 파드를 어떤 워커 노드에 생성할 것인지를 결정 및 할당.

## Worker Node

### 5). kubelet
> 파드의 구성 내용을 받아서 컨테이너 런타임으로 전달하고, 파드 안의 컨테이너들이 정상적으로 작동하는지 모니터링.

### 6). 컨테이너 런타임
> 파드를 이루는 컨테이너의 실행을 담당합니다. 파드 안에서 다양한 종류의 컨테이너가 문제 없이 작동하게 만드는 표준 인터페이스입니다. 

### 7). 파드
> 한 개 이상의 컨테이너로 단일 목적의 일을 하기 위해서 모인 단위입니다.
> 즉, 웹서버 역할을 할 수도 있고 로그나 데이터를 분석할 수도 있습니다.
> 언제라도 죽을 수 있는 존재.

### 11). 네트워크 플러그인
> 쿠버네티스 클러스터의 통신을 위해서 네트워크 플러그인을 선택하고 구성

### 12). CoreDNS
> 클라우드 네이티브 컴퓨팅 재단에서 보증하는 프로젝트, 빠르고 유연한 DNS 서버입니다.

---

# 3. 사용자가 배포된 파드에 접속할 때

#### 1). kube-proxy: 쿠버네티스 클러스터는 파드가 위치한 노드에 kube-proxy를 통해 파드가 통신할 수 있는 네트워크를 설정합니다. 

#### 2). 파드: 이미 배포된 파드에 접속하고 필요한 내용을 전달받습니다.

> 1. kubectl을 통해 API 서버에 파드 생성을 요청합니다.
> 2. (업데이트가 있을 때 마다 매번) API 서버에 전달된 내용이 있으면 API 서버는 etcd에 전달된 내용을 모두 기록해 클러스터의 상태 값을 최신으로 유지합니다. 따라서 각 요수가 상태를 업데이트할 때마다 모두 API 서버를 통해 etcd에 기록됩니다.
> 3. API 서버에 파드 생성이 요청된 것을 컨트롤러 매니저가 인지하면 컨트롤러 매니저는 파드를 생성하고, 이 상태를 API 서버에 전달합니다. 참고로 아직 어떤 워커 노드에 파드를 적용할지는 결정되지 않은 상태로 파드만 생성합니다.
> 4. API 서버에 파드가 생성됐다는 정보를 스케줄러가 인지합니다. 스케줄러는 생성된 파드를 어떤 워커 노드에 적용할 지 조건을 고려해 결정하고 해당 워커 노드에 파드를 띄우도록 요청합니다.
> 5. API 서버에 전달된 정보대로 지정한 워커 노드에 파드가 속해 있는지 스케줄러가 kubelet으로 확인합니다.
> 6. kubectl에서 컨테이너 런타임으로 파드 생성을 요청합니다.
> 7. 파드가 생성됩니다.
> 8. 파드가 사용 가능한 상태가 됩니다.

---

# 4. 오브젝트란
### * 스펙과 상태등의 값을 가지고 있는 파드와 디플로이먼트를 개별 속성을 포함해 부르는 단위 
 
 ---
 
#### 기본 오브젝트
> 파드
>> - 쿠버네티스에서 실행되는 최소 단위, 즉 웹 서비스를 구동하는데 필요한 최소 단위.
>> - 독립적인 공간과 사용 가능한 IP 소유.

> 네임스페이스
>> - 쿠버네티스 클러스터에서 사용되는 리소스들을 구분해 관리하는 그룹.

> 볼륨
>> - 파드가 생성될 때 파드에서 사용할 수 있는 디렉터리를 제공.

> 서비스
>> - 파드가 죽었는지 살았는지 신경 쓰지 않아도 이를 논리적으로 연결하는 것.
>> - 기존 인프라의 로드밸런서, 게이트웨이랑 비슷한 역할.

#### 디플로이먼트
> 기본 오브젝트만으로 쿠버네티스를 사용할 수 있습니다. 하지만 한계가 있어 이를 좀 더 효율적으로 작동하도록 기능들을 조합하고 추가해 구현한 것이다.

