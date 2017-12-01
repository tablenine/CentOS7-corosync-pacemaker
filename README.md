 # CentOS7에서 corosync와 pacemaker를 이용하여 이중화

 ## 0. 설치 전 확인사항

 + 각 노드의 IP주소를 /etc/hosts 파일에 등록 해준다

   ```
   cat /etc/hosts
   192.168.1.21  mf1
   192.168.1.22  mf2
   192.168.1.20  mf
   ```

 ## 1. 설치

 + yum으로 설치
 ```sh
     yum install -y pacemaker corosync pcs psmisc policycoreutils-python
 ```

 + 수동 설치
  ```sh
     rpm -Uvh cifs-utils-6.2-10.el7.x86_64.rpm 
     rpm -Uvh clufter-common-0.76.0-1.el7
     rpm -Uvh clufter-bin-0.76.0-1.el7.x86_64.rpm
     rpm -Uvh libqb-1.0.1-5.el7.x86_64.rpm
     rpm -Uvh resource-agents-3.9.5-105.el7_4.2.x86_64.rpm
     rpm -Uvh python-clufter-0.76.0-1.el7
     rpm -Uvh psmisc-22.20-11.el7
     rpm -Uvh --nodeps libsemanage-2.5-8.el7.x86_64.rpm
     rpm -Uvh libsemanage-python-2.5-8.el7.x86_64.rpm
     rpm -Uvh --nodeps policycoreutils-python-2.5-17.1.el7.x86_64.rpm
     rpm -Uvh policycoreutils-2.5-17.1.el7.x86_64.rpm
     rpm -Uvh --nodeps corosynclib-2.4.0-9.el7_4.2.x86_64.rpm
     rpm -Uvh corosync-2.4.0-9.el7_4.2.x86_64.rpm
     rpm -Uvh pacemaker-libs-1.1.16-12.el7_4.4.x86_64.rpm
     rpm -Uvh pacemaker-cli-1.1.16-12.el7_4.4.x86_64.rpm
     rpm -Uvh pacemaker-cluster-libs-1.1.16-12.el7_4.4.x86_64.rpm 
     rpm -Uvh pacemaker-1.1.16-12.el7_4.4.x86_64.rpm
     rpm -Uvh pcs-0.9.158-6.el7.centos.x86_64.rpm
  ```

+ 추가 설치 필요 할 수도 있음

```sh
  rpm -Uvh libyaml-0.1.4-11.el7_0.x86_64.rpm
  rpm -Uvh ruby-irb-2.0.0.648-30.el7.noarch.rpm
  rpm -Uvh rubygem-io-console-0.4.2-30.el7.x86_64.rpm
  rpm -Uvh rubygem-psych-2.0.0-30.el7.x86_64.rpm
  rpm -Uvh rubygem-rdoc-4.0.0-30.el7.noarch.rpm
  rpm -Uvh rubygems-2.0.14.1-30.el7.noarch.rpm
```

  ​

  ## 2. selinux, 방화벽 설정

 + 방화벽 설정
 ```sh
     firewall-cmd --permanent --zone=public --add-port=2224/tcp
     firewall-cmd --permanent --zone=public --add-port=3121/tcp
     firewall-cmd --permanent --zone=public --add-port=21064/tcp
     firewall-cmd --permanent --zone=public --add-port=5405/udp
     firewall-cmd --reload
 ```
 + selinux 해제
   + selinux 상태 확인
     ```sh
       sestatus
     ```

   + selinux 해제

     ```sh
     sentenforce 0
     ```
   + selinux 영구 해제
     ```sh
     vi /etc/sysconfig/selinux
     SELINUX=enforcing -> SELINUX=disabled 로 변경
     ```



 ## 3. 설정 

 + 데몬 실행 및 서비스 추가

   + 데몬 실행(모든 노드 실행)

     ```sh
     systemctl start pcsd.service
     ```

   + 서비스 추가(모든 노드 실행)

     ```sh
     systemctl enable pcsd.service
     systemctl enable corosync.service
     systemctl enable pacemaker.service
     ```

 +  hacluster 계정 패스워드 설정 (모든 노드를 동일하게 설정)

   ```sh
   passwd hacluster
   ```

 + cornsync 설정

   + hacluster 사용자 인증(한쪽 노드)

     ```sh
     pcs cluster auth mf1 mf2
     ```

   + corosync 구성(한쪽 노드)

     ```sh
     pcs cluster setup --name mf_cluseter mf1 mf2
     ```



 ## 4. cluster 실행과 상태 확인 

 + 클러스터 실행

   ```sh
   pcs cluster start -all
   ```

   결과

   ```
   mf1: Starting Cluster...
   mf2: Starting Cluster...
   ```

 + 클러스터 통신 확인

   ```sh
   corosync-cfgtool -s
   ```

   결과

   ```
   Printing ring status.
   Local node ID 1
   RING ID 0
   	id	= 192.168.1.21
   	status	= ring 0 active with no faults
   ```

 + 멤버쉽과 쿼럼 확인

   ```sh
   corosync-cmapctl |egrep -i members
   ```

   ``` sh
   pcs status corosync
   ```

   ```sh
   pcs status
   ```



 ## 5. Active/Passive 클러스터 생성

 + 유효성 확인 

   데이터의 무결성 확보를 위해 기본적으로 STONITH가 활성되어 있으나 비활성화 후 확인

   ```sh
   pcs property set stonith-enabled=false
   crm_verify -L
   ```

 + 가상 IP리소스 생성

   ```sh
   pcs resource create VirtualIP ocf:heartbeat:IPaddr2 ip='사용할 가상 IP' cidr_netmask=24 op monitor interval=30s
   ```



 ## X. constraint location 설정

 + location 추가

   숫자가 높을 수록 우선순위 높음

   ```sh
   pcs constraint location VirtualIP prefers mf1=50
   pcs constraint location VirtualIP prefers mf2=100
   ```

 + location 삭제

   + id 확인

     ```
     pcs constraint --full
     ```

     결과

     ```
     Location Constraints:
       Resource: VirtualIP
         Enabled on: mf2 (score:100) (id:location-VirtualIP-mf2-100)
         Enabled on: mf1 (score:50) (id:location-VirtualIP-mf1-50)
     Ordering Constraints:
     Colocation Constraints:
     Ticket Constraints:
     ```

   + 삭제

     ```sh
     pcs constraint remove `삭제 할 constraint id`
     ex)location-VirtualIP-mf2-100
     ```


 

## X. 테스트

+ standby 상태로 변경

```sh
pcs cluster standby mf1
```

+ standby 상태에서 복구

```sh
pcs cluster unstandby mf1
```

 ### 출처 

 * [리눅스 HA(corosync, pacemaker) – Part 1](https://blog.boxcorea.com/wp/archives/1784)
 * [PCS를 이용한 HA구성](https://yoanp.github.io/2017/03/04/pcs-ha-setting)
