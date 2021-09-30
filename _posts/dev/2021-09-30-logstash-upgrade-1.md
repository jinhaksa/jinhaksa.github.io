---
title: Logstash Upgrade
categories: [dev]
tags:
  - logstash
  - upgrade
  - 7
description: 로그스태시 7버전 업그레이드
---

<!-- @format -->

# 로그스태시 업그레이드(1)

## Elasticsearch 7.0 주요 변경 사항 정리

Elasticsearch 7.0 이 2019년 4월에 출시되었다.  
현재 사용하고 있는 엘라스틱서치 6 버전에서 업그레이드를 하기 전 주요 변경사항을 공식 도큐먼트롤 보며 정리해봤다.

모든 변경 사항들이 중요하겠지만, 실질적으로 사용하고 있고 변경 사항부터 정리했다.

## Analysis changes

### \_analyze로 생성할 수 있는 토큰 수 제한

\_analyze를 통해 생성할 수 있는 토큰의 수를 10,000개로 제한  
제한 없는 토큰의 수로 메모리 부족을 방지하기 위함

### standard 필터 제거

standard 필터는 analyze에서 변경되는 값이 없기 때문에 **제거**되었습니다.

### standard_html_strip analyzer 지원 중단

standard_html_strip 분석기는 지원이 중단되었으므로, standard(tokenizer) + html_strip(char_filter)로 대체하여야 합니다.  
(standard_html_strip 를 사용한 인덱스는 7.0에서 계속 읽을 수 있지만 이를 사용한 새 인덱스는 생성할 수 없습니다.)

### nGram edgeNGram 토큰 필터는 새로운 인덱스에서 사용 중단

6.x에서 사용되던 nGram과 edgeNGram 토큰 필터 이름은 ngram과 edge_ngram 으로 변경하여 사용해야합니다.  
(7.0에서 계속하여 읽기는 가능)

### ngramTokenFilter 및 ngramTokenizer 에서의 max_size와 min_size 제한

NGramTokenFilter, NGramTokenizer 의 max_ngram/min_ngram가 1로 제한 되었습니다.  
해당 제한은 index.max_ngram_diff 설정으로 변경할 수 있습니다.

---

## Cluster changes

### 샤드 기본 설정 중 \_primary, \_primary_first, \_replica, \_replica_first 제거됨

### 운영 배포 때에는 elasticsearch.yml 파일에 아래의 항목 중 하나 이상을 설정해야 함

- `discovery.seed_hosts`(only 7)
- `discovery.seed_providers`(only 7)
- `cluster.initial_master_nodes`(only 7)
- `discovery.zen.ping.unicast.hosts`
- `discovery.zen.hosts_provider`

(이전 버전에서 업그레이드하는 경우, discovery.zen.ping.unicast.hosts 또는discovery.zen.hosts_provider 설정 필요)

### 요청의 타임아웃 시간 감소

기존 : ping 30초 3번(90초)을 응답하지 않으면 클러스터 제거  
변경 : ping 10초 3번(30초)을 응답하지 않으면 클러스터 제거

---

## Indices changes

### 인덱스 생성 옵션 중 샤드의 기본값 변경

기존 인덱스의 샤드 기본 값이 5 에서 1로 변경

### 문서 배포 변경 사항

7.0 이상에서 생성된 인덱스는 index.number_of_routing_shards 값이 설정됨 (샤드에 분산되는 데이터 방식)  
이전 버전과 동일하게 유지하려면 index.number_of_routing_shards를 index.number_of_shards로 설정해야 합니다.

## Mapping changes

### \_all 메타 데이터 필드 제거

### \_uid 메타 데이터 필드 제거

기존 \_uid 는 \_type + \_id로 구성된 복합키를 인덱싱하는데 사용했습니다.  
이제 인덱스는 여러 type을 가질 수 없으므로 \_id 로 대체 됩니다.

### \_default\_ 매핑 사용할 수 없음

7.0에서 인덱스, 인덱스 템플릿에 _default_ 매핑이 사용된 경우 인덱스를 생성하지 못함  
(\_default\_ 제거 필요)

### nested json 객체의 수 제한

nested json 객체 수를 10000으로 제한합니다.(메모리 부족 방지를 위함)  
index.mapping.nested_objects.limit 로 변경 가능합니다.

### 지원되지 않는 옵션을 사용할 경우 유사도 검색에서 오류 발생

이전에는 알 수 없는 매개변수가 있는 경우 무시되었지만, 7.0부터는 오류 발생

### 아래 geo_shape필드 타입의 파라미터 사용 중단

tree, precision, tree_levels, distance_error_pct, points_only, strategy

(향후 버전에서 제거될 예정)

### include_type_name의 기본값 false로 변경

7.0부터 여러개의 타입을 생성할 수 없기 때문에 include_type_name을 true로 줬을 때 deprecation warning 메시지 출력  
(8.0에서 제거)

---

## Search and Query DSL changes

### Changes to queries

- fuzzy 쿼리의 transpositions 파라미터 기본값 true로 변경
- query_string 옵션에서 use_dismax, split_on_whitespace, all_fields, locale, auto_generate_phrase_query, lowercase_expanded_terms 제거
- MUST_NOT 절의 스코어가 1에서 0으로 변경됐습니다.
- span query 내부에 boost를 사용하면 exception 발생
  (span query : )

### Adaptive replica selection가 기본적으로 활성화

ARS를 기본적으로 활성화  
(ARS : primary shard에서만 데이터를 읽는 것이 아닌 replica에서도 데이터를 읽을 수 있도록 하는 기법)

```jsx
# 비활성화
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.use_adaptive_replica_selection": false
  }
}
```

### terms query 요청에서 사용할 수 있는 terms의 개수 제한

최대 terms 개수를 65536으로 제한  
많은 terms를 사용할 경우, 사용한 만큼의 메모리가 필요하여 클러스터 성능이 저하될 수 있음

```jsx
#terms query 사용 예
{
  "query" : {
    "bool" : {
      "filter" : [
        {
          "terms" : {
            "ipAddressField" : [
              "100.100.100.1",
              "100.100.100.2",
              "100.100.100.3",
							....
            ],
            "boost" : 1.0
          }
        }
      ],
      "adjust_pure_negative" : true,
      "boost" : 1.0
    }
  }
}
```

### Regexp 쿼리 요청에서 사용할 수 있는 regex의 길이 제한

정규식의 최대 길이를 1000으로 제한  
해당 기본값은 index.max_regex_length 설정으로 변경 가능

(긴 정규식 문자열을 사용한 정규식 쿼리는 검색 성능이 저하될 수 있음)

### auto-expanded fields 수 제한

all fields 모드("default_field": "\*") 의 수를 최대 1024로 제한  
indices.query.bool.max_clause_count 옵션을 사용하여 최대 수 변경 가능  
(auto-expanded fields : query_string, simple_query_string 등등)

### max_score가 트래킹되지 않을 때 null로 세팅

기존에는 max_score가 추적되지 않을 때 0으로 설정되었던 값을 null 로 설정

### 음수를 사용한 부스터는 허용되지 않음

기존에 부스터에서 사용하던 음수는 더 이상 허용되지 않음  
(부스트의 값 : 0~1 사이)

### Function score Query 에서 음수 스코어가 허용되지 않음

Function score query에서 음수를 사용할 경우 오류 발생

### hit.total이 object로 변경

```jsx
# 기존
"hits" : {
    "total" : 4675,
    "max_score" : 1.0,
		...
}

# 7.0
"hits" : {
    "total" : {
			"value": 4675,
			"relation": "eq"
		}
    "max_score" : 1.0,
		...
}
```

relation이 eq일 경우, 정확한 개수라는 의미  
rest_total_hits_as_int=true 옵션으로 숫자로 변경 가능

### track_total_hits 기본값은 10,000

기본적으로 검색 요청에 대해 10,000개의 도큐먼트까지는 정확한 카운트를 출력  
검색 결과가 10,000이 넘어갈 경우에는 10,000으로 보여주며 relation을 gte로 표기

```jsx
# 검색 결과가 10,000개가 넘었을 경우
{
  "_shards": ...
  "timed_out": false,
  "took": 100,
  "hits": {
    "max_score": 1.0,
    "total": {
      **"value": 10000,
      "relation": "gte"**
    },
    "hits": ...
  }
}
```

track_total_hits: true 로 세팅하여 정확한 값을 강제할 수 있음

```jsx
GET /kibana_sample_data_flights/_search
{
	"track_total_hits": true
}
```

참고자료

[https://www.elastic.co/guide/en/elasticsearch/reference/current/breaking-changes-7.0.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/breaking-changes-7.0.html)

[https://www.elastic.co/guide/en/elasticsearch/reference/current/span-queries.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/span-queries.html)

[https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-fuzzy-query.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-fuzzy-query.html)

[https://jaimemin.tistory.com/m/1880?category=1216055](https://jaimemin.tistory.com/m/1880?category=1216055)
