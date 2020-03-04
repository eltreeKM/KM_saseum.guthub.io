---
layout: post
title: How to recover red state index
subtitle: red 상태가 되는 이유와 해결법
tags: [elasticsearch]
comments: true
---

## Status ##

elasticsearch의 state에는 3가지가 있다.

  1. green: 모든 샤드가 할당된 상태
  2. yellow: 모든 primary shard가 할당 되어 있지만 1개나 그 이상의 replica shard는 할당이 되지 않은 상태. 만약 node가 cluster fails 상태일 때 해당 노드의 데이터는 노드가 고쳐질 때 까지 산출되지 않을 수 도 있다.
  3. red: 1개 혹은 그 이상의 primary shard가 할당 되어 있지 않고, 데이터의 일부가 산출되지 않는 상태, primary shard 가 할당 될 때 잠시 일어날 수도 있다.
  
위 상태중 green 과 yellow 상태는 데이터의 읽기/쓰기에는 문제가 없지만 red 상태는 all shard failed 에러가 뜨며 아무것도 할 수 없는 상태가 된다.

나의 경우는 인덱스의 replica를 0으로 설정해놨기 때문에 primary shard가 할당이 되지 않은 상태였다.

elasticsearch는 primary shard를 할당할때 5번의 시도를하고 5번의 시도 후엔 할당을 하지 않는다. 

해결방법은 2가지가 있다.

1. 인덱스 삭제: red 인덱스를 삭제하면 문제가 사라진다.

2. 5번의 시도 기록을 지운다. 


2번의 선택이 더 매력적이기 마련이다

아주 간단히 kibana의 dev tool에서 

```
/_cluster/reroute?try_failed
```

를 실행 시키면 try_failed 가 초기화 되면서 primary shard 가 자동 할당 된다.



