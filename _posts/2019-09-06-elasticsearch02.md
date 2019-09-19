---
layout: post
title: Bulk search with python
subtitle: How to search more than 10000 docs
tags: [elasticsearch, searching, python]
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
