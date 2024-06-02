# Elastic Search


### 특징

* ES 는 기본적으로 검색엔진이다.
    * **Apache Lucene** 이라는 검색 라이브러리를 핵심 라이브러리로 사용
* Full-text Search 를 지원하며, 다양한 종류의 검색 쿼리를 지원.
* Analyzer 를 조합하여 색인 구성 및 형태소 분석 가능
* JSON 형태의 문서를 저장, 색인, 검색
* Near real-time search 를 지원

### 기본 용어

| 용어 | 의미 |
| --- | --- |
| 문서 | 저장, 색인을 생성하는 JSON 문서 |
| 인덱스 | 문서를 모아놓는 단위, Client 는 인덱스 단위로 검색 요청 |
| 샤드 | 인덱스를 여러 샤드로 분리하여 저장, HA 제공을 위해 이 샤드를 복제 |


* 문서를 분석하여 역색인을 하는 주체는 Apache Lucene
* ES 는 이 루씬을 이용하여 사용자에게 하여금 HA, 분산처리를 지원하고 다양한 API 를 제공

### 동작 방식

#### 루씬 Flush
1. 루씬은 문서 색인 요청이 들어오면, 문서를 분석하여 역색인을 생성
2. memory buffer의 분석 내용을 주기적으로 디스크에 flush

#### 루씬 Commit
* 루씬의 Flush 는 page cache 에 넘겨주는 것까지만 보장
* Commit 이 디스크에 안전히 저장되도록 보장해줌
* fsync 를 통해 주기적으로 page cache - disk 내용 싱크를 맞춤

루씬은 역색인 정보를 파일로 저장한다. 이때, 파일을 연 시점에서 색인이 완료된 문서만 검색할 수 있다. `Near real-time search` 는 이러한 동작 방식으로 데이터를 추가한 후에 바로 검색 결과에 반영될 수 없는 특징으로 나타나는 현상이다.

#### ES Refresh
ES 는 변경 내용을 검색에 반영하기 위해, `DirectoryReader.openIfChanged` 를 통해 파일을 주기적으로 다시 연다. 이를 refresh 라고 한다.

#### Segment
* Lucene 의 검색 대상
* Lucene Flush -> Lucene Commit 으로 디스크에 기록된 파일이 모이면 Segment 라는 단위가 된다.
* immutable 하다는 특징이 있음
    * 새로운 문서가 들어오면 새 새그먼트가 됨
    * 삭제를 할 경우, 삭제 플래그만 표시
    * 업데이트 발생 시, 삭제 플래그를 표시하고, 새 세그먼트를 형성
    * 주기적으로 병합하며, 이때 삭제 플래그된 세그먼트는 실제로 삭제함

TODO: 5GB 보다 큰 세그먼트는 추가적인 데이터 변경이 일어나면 추가된 작은 세그먼트는 병합 대상 선정에서 영원한 누락의 가능성이 있음

#### Lucene Index
* 여러 세그먼트가 모이면 하나의 루씬 인덱스
* ES shard 는 이 루씬 인덱스 하나를 래핑한 것
* ES shard 가 여러개가 모이면 ES 인덱스가 된다.
* 이러한 구조로, Client 는 여러 샤드가 있는 문서를 모두 검색할 수 있다.
* ES shard는 여러 노드에 분산되어 있고, 이런 노드들이 모여 하나의 ES cluster 가 된다.

#### 문제 상황
ES 에 색인된 문서는 lucene commit 까지 완료해야 데이터에 안전하게 기록
* 변경사항이 생길때마다 commit 하기엔 비용이 너무 큼
* 변경사항을 모아서 commit 하기에는 장애 발생시 데이터 복구가 힘듦

#### Translog
* ES shard 는 작업마다 translog 라는 로그를 색인, 삭제 작업이 lucene index에 수행된 직후에 기록
* 파일에 데이터가 저장 된것이 보장되면, lucene commit 이 끝나면 translog 를 삭제
* 그래서, 이 translog 에 fsync가 됐을때 Client 에 CRUD가 성공했다고 함

## Text

ES 는 앞서 언급했다시피, Full-text Search 를 지원한다.

### Text
Text 는 ES에서 지원하는 자료형 중 하나로 ES 는 Text 자료형은 Anaylizer가 적용된 후에 색인한다.

#### Analyzer
* 0개 이상의 character filter
* 1개의 tokenizer
* 0개 이상의 token filter 로 구성

#### Character Filter
* text를 character stream 으로 받아와 문자를 추가, 변경, 삭제
* 여러 built-in character filter 가 있음
    * HTML strip character : HTML tag 안의 데이터를 꺼내고 디코딩 한다.
    * mapping character filter : key:value 식으로 전달하여 문자를 치환
    * pattern replace : regex 를 이용하여 치환

#### Tokenizer
* standard tokenizer : [Unicode Text Segmentation Algorithm](https://unicode.org/reports/tr29/) 이용, 대부분의 문장 부호와 whitespace 를 기준으로 나눈다.
* keyword tokenizer : 쪼개지 않는다. (Analyzer 는 1개 이상의 tokenizer 로 구성되어있기 때문에 존재)
* ngram tokenizer : [min_gram, max_gram] 의 단위로 쪼갠다.
* edge_ngram : [min_gram, max_gram] 의 길이를 갖으며, 첫 글자를 포함하는 토큰을 생성한다.
