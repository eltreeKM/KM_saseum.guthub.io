---
layout: post
title: Bulk search with python
subtitle: How to search more than 10000 docs
tags: [elasticsearch, searching, python3]
comments: true
---
엘라스틱서치에서 search를 진행하면 size는 default 10으로 설정이 되있고 최대 10000개까지 출력이 가능함

하지만 10000개 이상의 문서를 뒤져보고싶을땐?

크게 2가지를 찾았다.

1.from과 size 이용

2.scroll 이용

이 중 scroll에 대해 정리 하였음

## Scroll 이란?

While a search request returns a single “page” of results, the scroll API can be used to retrieve large numbers of results (or even all results) from a single search request, in much the same way as you would use a cursor on a traditional database.

출처: https://www.elastic.co/guide/en/elasticsearch/reference/7.3/search-request-body.html#request-body-search-scroll

해석하면, 검색 요청이 단일 '페이지'를 리턴할 때, scroll API를 사용하면 수 많은 결과(혹은 모든 결과)를 단일 검색 요청으로 리턴할 수 있다. 전통적인 데이터베이스의 '커서'와 같은 역할이다. 라고 번역할 수 있음.

scroll api를 사용하면 설정한 시간만큼 id가 활성화 되며 이 scroll id를 통해 10000개 이후의 데이터를 중복없이 조회 할 수 있다.

또 한, scroll은 실시간 요청이라기 보단, 인덱스의 지금 상태를 메모리에 저장 해 자주 바뀌는 인덱스에 대한 검색질의 결과가 달라 지는것을 방지할 수 있다. 너무 많이 사용되면 메모리 부족을 야기할 수 있으므로 scrol timeout 옵션을 사용해 메모리에 남아있는 기간을 정할 수 있다.

아래 예제 부터는 elasticsearch python module을 이용한다. pyrhon의 버전은 3.6

예제에 쓰일 데이터의 매핑은 아래와 같다.
~~~
20190820의 네이버카페 글을 크롤링한 정보로써 제목, 본문, 포스트를 올린 날짜, 시간으로 이루어져있다.

{
  "20190820" : {
    "mappings" : {
      "properties" : {
        "@timestamp" : {
          "type" : "date"
        },
        "@version" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "description" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "pubdate" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "pubtime" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "title" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
    }
  }
}
~~~

일반적인 search api로 검색을 진행 했을 경우 10000개 이상의 결과를 반환하게 하면 아레와 같은 에러가 발생한다.
~~~
def normal_search(es_client):
        return(
            es_client.search(
                index = '20190820',
                body = {
                    'size': 100001,
                    'query': { 'match_all': {}}
                }
            )
        )


elasticsearch.exceptions.RequestError: RequestError(400, 'search_phase_execution_exception', 'Result window is too large, from + size must be less than or equal to: [10000] but was [100001]. See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the [index.max_result_window] index level setting.')
~~~

index.max_result_window 세팅을 바꾸라는 에러 메시지가 출력된다

다른 설정을 건드리지 않으면 10001개 이상의 결과를 뽑아내는것은 불가능 하다. 내가 조회해야 할 인덱스의 문서를 확인해 보자

~~~
curl -XGET 'localhost:9200/_cat/indices?v&pretty'

health status index      uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   20190820   ETyYk0wLQMKiNcL4_g1GzA   1   1     108839        10000    511.8mb        255.9mb
~~~

무려 108839개의 문서가 있다. 모든 문서를 조회해 보자, 스크롤은 search API를 통해 사용 할 수 있고, 아래와 같이 사용한다.

~~~
def scroll_api(self, es_client):
        result = es_client.search(
            index = '20190820',
            scroll = '2m',
            size = 10000,
            body = {
                'query' : { 'match_all': {}}
            }
        )

        scroll_id = result['_scroll_id']
        scroll_size = len(result['hits']['hits'])

        print('scroll start')
        print(scroll_id)
        print(scroll_size)

        while(scroll_size >0):
            result = es_client.scroll(scroll_id = scroll_id, scroll = '2m')
            scroll_id = result['_scroll_id']
            scroll_size = len(result['hits']['hits'])

            print(scroll_id)
            print(scroll_size)

        print('scroll complete')
        
        

scroll start
DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFMQWRHRHT000WnJSbk93WnJGNkwzcXMtZw==
10000
DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFMQWRHRHT000WnJSbk93WnJGNkwzcXMtZw==
10000
DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFMQWRHRHT000WnJSbk93WnJGNkwzcXMtZw==
10000
DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFMQWRHRHT000WnJSbk93WnJGNkwzcXMtZw==
10000
DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFMQWRHRHT000WnJSbk93WnJGNkwzcXMtZw==
10000
DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFMQWRHRHT000WnJSbk93WnJGNkwzcXMtZw==
10000
DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFMQWRHRHT000WnJSbk93WnJGNkwzcXMtZw==
10000
DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFMQWRHRHT000WnJSbk93WnJGNkwzcXMtZw==
10000
DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFMQWRHRHT000WnJSbk93WnJGNkwzcXMtZw==
10000
DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFMQWRHRHT000WnJSbk93WnJGNkwzcXMtZw==
10000
DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFMQWRHRHT000WnJSbk93WnJGNkwzcXMtZw==
8839
DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAFMQWRHRHT000WnJSbk93WnJGNkwzcXMtZw==
0
~~~

총 108839개의 문서를 반환했으며, 데이터베이스의 '커서' 역할을 하니 중복없이 출력이 되었다.

앞에서 말했다시피 scroll은 메모리영역에 저장되어서 관리를 잘 안해줄경우 메모리가 부족하게 되는 현상이 발생 할 수도 있다.
scroll id 를 이용한 아래와 같은 방법으로 scroll을 삭제할 수 있다.

~~~
curl -X DELETE "localhost:9200/_search/scroll?pretty" -H 'Content-Type: application/json' -d'
{
    "scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAACSxsWdGkzekl6QnFSUTY4VHJZMGZLeFF6Zw=="
}
'


{
  "succeeded" : true,
  "num_freed" : 1
}
~~~
