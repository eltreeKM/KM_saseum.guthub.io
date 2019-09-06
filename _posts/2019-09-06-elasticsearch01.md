---
layout: post
title: Elasticsearch Install & configuration
subtitle: How to build cluter
tags: [elasticsearch, configuration]
comments: true
---

물리서버 3대에 엘라스틱서치 클러스터를 구성할 일이 생김

구성하면서 생긴 오류와 그 해결법

포스팅 순서는 서버설정 -> elasticsearch.yml -> kibana.yml -> 실행  

**구성**

각 서버에는 하나의 마스터 노드, 그 아래 데이터노드를 구성해 하나의 클러스터를 구현

**환경**

* elasticsearch 7.3.0
* kibana 7.3.0
* 스토리지 용량이 500GB, 메모리가 8기가 동일스펙인 물리서버 1.1.1.1, 1.1.1.2, 1.1.1.3 (회사에서 사용하는 서버라 임의의 host adress로 대체)
* centos 6.10


## 설치

yum 으로 설치해도 되지만 회사에서 권장하지 않아 filezilla를 이용해 직접 tar.gz를 넣는 방식으로 진행
클러스터 이름은 test-cluster

각 마스터 노드의 이름은 호스트넘버를 따서 master-1, master-2, master-3으로 지정, 9200번 port를 http port로, 9300번 port를 transport port로 지정

각 데이터 노드의 이름은 호스트넘버를 가운데 넣어서 node-1-n, node-2-n, node-3-n 으로 지정, 920n번 port를 http port로, 9300번 port를 transport port로 지정

**오류와 해결**


1.Exception java.lang.RuntimeException: max file descriptors [65535] for elasticsearch process likely too low, increase to at least [65536] 오류
클러스터를 사용하기 위해선 리소스 사용에 대한 제한을 풀어줘야하기 떄문에 생기는 오류

~~~
unlimit -Sa
~~~

커맨드로 현재 리소스 제한 현황을 확인 가능

**해결**
  
아래 config는 root 권한으로 진행

~~~
vi /etc/security/limits.conf

elasticsearch hard memlock unlimited
elasticsearch soft memlock unlimited
elasticsearch hard nofile 65536
elasticsearch soft nofile 65536
elasticsearch hard nproc 65536
elasticsearch soft nproc 65536
~~~

위 코드에서 elasticsearch는 Elasticsearch를 실행하게 되는 리눅스 유저이름

서버를 reboot 해준다

2.max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

프로세스가 가질 수 있는 메모리 맵의 수를 늘려야 한다는 오류
root 권한을 획득 한 후 아래 커맨드 입력

**해결**
  
~~~
sysctl -w vm.max_map_count=262144 
~~~


3.포트 개방 

포트 설정을 건드리지 않고 실행을 하게 되면 유저가 접근하기 위한 http port는 9200, 노드끼리 통신하기 위해 사용되는 transport port는 9300으로 설정됨
서버끼리 통신하기 위해선 마스터서버의 transport port인 9300번을 개방해야함, 개방하지 않고 엘라스틱서치를 실행 시 각 서버에 test-cluster라는 이름의 
클러스터가 각각 생겨나게됨.. 

**해결**
  
root 권한 획득 후

~~~
vi /etc/sysconfig/iptables
-A INPUT -m state --state NEW -m tcp -p tcp --dport 9300 -j ACCEPT
-A INPUT -m state --state NEW -m tcp -p tcp --dport 5601 -j ACCEPT #키바나를 위해 미리 열도록 하자
~~~

위 코드를 추가해야하는데 순서가 중요하다.

~~~
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
~~~

위 코드 뒤에 추가하면 열리지 않았다. 파일을 위부터 순서대로 읽어서 모든 포트를 reject한다는 명령이 먼저 이루어 져서 그런것 같음
  
위 코드 위에 추가 해 주도록 하자.
  
저장 후, iptables 서비스를 재시작

~~~
service iptables restart
~~~

4.elasticsearch.yml - master

~~~
vi config/elasticsearch.yml

cluster.name: custom-dashboard
node.name: master92
node.master: true
node.data: false
node.ingest: true
path.data: 데이터저장경로
path.logs: 로그저장경로
bootstrap.system_call_filter: false
network.host: 0.0.0.0
discovery.seed_hosts: ["1.1.1.1", "1.1.1.2", "1.1.1.3"]
cluster.initial_master_nodes: ["master1", "master2", "master3"]
transport.port: 9300
http.port: 9200
transport.tcp.compress: true
http.cors.enabled: true
xpack.monitoring.enabled: true
xpack.monitoring.collection.enabled: true
~~~

- bootstrap.system_call_filter: false => false로 해주지 않으면seccomp를 사용하지 않는 서버에서 오류가 나게 됨
- network.host: 0.0.0.0 => network.host로 ip를 설정하게 되면 bind_host와 publish_host 둘 다 같은 ip로 설정이 됨
- discovery.seed_hosts => 엘라스틱서치 실행 시 다른 네트워크에 있는 노드를 검색하는 행위를 discovery라고 하며, 포트번호를 적지 않을 시 9200~9299사이의 값을 자동으로 찾아 실행되고있는 노드를 감지한다.
- cluster.initial_master_nodes => 엘라스틱서치 실행 시 반드시 실행되고있어야 할 마스터 노드의 이름을 적는다. 
- node.ingest: true => mornitoring 기능을 켜려면 ingest노드가 되어야 하는데 이는 더 알아보고 나중에 포스팅.

5.elasticsearch.yml - data node 
아래 항목을 제외하곤 master와 동일하게 수정

~~~
node.master: false
node.data: true
node.ingest: false
transport.port: 9301
http.port: 9201
~~~
  
