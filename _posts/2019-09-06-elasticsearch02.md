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
