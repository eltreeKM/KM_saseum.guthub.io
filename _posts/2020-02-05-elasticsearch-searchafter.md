---
layout: post
title: Search-after
subtitle: How to search more than 10000 docs 
tags: [elasticsearch, search]
comments: true
---

지난번 포스팅에서 scroll을 알아보았는데 scroll에는 치명적인 단점이 있다.
1.scroll 정보는 search context 라는 정보로 메모리에 저장되기 때문에 다른 작업을 수행 할 시 여러개를 사용하면 메모리가 부족할 수 있다.
2.1의 이유로 다른 작업을 수행 할 시 timeout이 빈번하게 발생 할 수 있다.

실제로 회사에서 scroll을 이용해 150만건의 데이터를 조회하는 동안 다른 쿼리를 입력했을 시 request_timout 에러가 많이 발생하였다.
그래서 elasticsearch 커뮤니티에 질문 결과 많은 대답을 받았는데, 그 중 하나가 메모리 낭비를 줄이기 위해 search_after를 사용하는것이었다.

search_after를 알아보도록 하자

**search_after 란?**

Pagination of results can be done by using the from and size but the cost becomes prohibitive when the deep pagination is reached. The index.max_result_window which defaults to 10,000 is a safeguard, search requests take heap memory and time proportional to from + size. The Scroll api is recommended for efficient deep scrolling but scroll contexts are costly and it is not recommended to use it for real time user requests. The search_after parameter circumvents this problem by providing a live cursor. The idea is to use the results from the previous page to help the retrieval of the next page.

출처: https://www.elastic.co/guide/en/elasticsearch/reference/7.5/search-request-body.html#request-body-search-search-after

엘라스틱서치 공식 문서에는 search_after는 from, size를 이용하는 방법과 scroll api를 이용할 떄의 리소스 낭비를 피하기 위해 라이브 커서를 제공한다고 한다.
검색 결과를 이용해 다음 검색을 한다는 것이다.

실습 환경
* elasricsearch-7.5.2
* kibana-7.5.2

~~~
사용데이터는 2월 1일 네이버 댓글을 크롤링 하였고, 이에 랜덤으로 id번호를 부여했다.
현재 20200201의 인덱스에는 1639880개의 문서가 존재한다.

GET /20200201/_search
{
  "size":0,
  "sort": [
    {
      "id": {
        "order": "asc"
      }
    }
  ],
  "query": {
    "match_all": {}
  }
}


result
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10000,
      "relation" : "gte"
    },
    "max_score" : null,
    "hits" : [ ... ]
  }
}
~~~
id 순으로 오름차순으로 정렬을 해 검색을 실시한 결과 10000건의 결과가 나오게 되며 가장 마지막 문서의 id는 20807132505 이다.
search_after는 이 id를 이용해 이 아이디보다 큰 다음 10000개의 검색결과를 가져오는 방식으로 진행한다.

~~~
search_after 의 값엔 여러 값은 nested 형태로 여러개의 input을 입력할 수 있다.

GET /20200201/_search
{
  "search_after": [20807132505],
  "size": 10000, 
  "sort": [
    {
      "psid": {
        "order": "asc"
      }
    }
  ],
  "query": {
    "match_all": {}
  }
}
~~~

검색 결과 맨 처음 문서의 id는 20807132506 이고 마지막 문서의 id는 20807186715로 중복없이 검색이 되었다.

1만건이 넘는 문서를 색인하는동안 다른 여러개의 작업을 동시에 하고싶으면 scroll 대신에 search_after를 사용하면 좋다!ㅎ
