---
layout: post
title: 6 not so obvious things about Elasticsearch
subtitle: 엘라스틱서치에 대해 알아야 할 것 6가지
tags: [elasticsearch, analyzer, Nori]
comments: true
---

출처: https://blog.softwaremill.com/6-not-so-obvious-things-about-elasticsearch-422491494aa4

1. Elastic Stack
  - Elasticsearch
  - Kibana: 데이터 분석, 시각화
  - Logstash: 서버사이드 데이터 프로세싱 파이프라인
  - Beats: 단일목적 데이터 전송기
  - Elastic Cloud: 엘라스틱서치 클러스터를 호스팅해주는 클라우드
  - Machine Learning
  - APM: Application Performance Monitoring
  - Swiftype: 엘라스틱서치를 기반으로 하는 검색 기능 알고리즘

2. 2종류의 데이터 셋

 Static data and time series data
 
 - static data: 변화가 거의 없는 데이터, 일반 데이터베이스에 저장한 데이터로 생각 할 수 있다.
 - time series data(시계열 데이터): 모니터링같은 순간 변화가 큰 데이터와 관련된 데이터
 
 저장하는 데이터 유형에 따라 클러스터를 다른 방식으로 모델링 해야하며, static data의 경우 고정된 수의 인덱스와 샤드를 선택해야한다. 
 항상 데이터 셋의 모든 문서를 검색할 때 유리함, 
 tile-series 데이터의 경우 time binding rolling index를 선택해 최근의 데이터를 조회하고, 지난 데이터들은 삭제하거나 필요한 부분만 남기고 삭제
 
3. Search Score
  
  엘라스틱의 search 결과는 tf-idf 알고리즘을 기반으로 산출된다.
  예를 들면 아래와 같은 2개의 document가 있다고 치자:
   
    1. 나는 밥을 먹는 중이다.
    2. 우리는 일을 하는 중이다.
  
  '밥' 이라는 키워드에 대해 TF를 구해보면
    1. 조사를 제외한 4개의 단어 개수중 밥 과 일치하는 단어는 1개이므로 1/4
    2. 위와 같은 방법으로 구하면 0/4
  
  반면 IDF는 전체 데이터셋에 대해 단일값으로 계산이 된다.
  
  IDF는 [밥을 포함하고 있는 문서:모든문서]의 비율이며 위의 예의 경우 다음과 같은 결과가 도출된다.
  ```
  log(2/1) = 0.301
  ```
