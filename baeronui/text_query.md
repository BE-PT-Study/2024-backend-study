# Text Query

DB에 저장해놓은 text를 검색하기 위해서는 간단하게 2가지 방법을 생각할 수 있다.
1. 기 사용하던 RDB (MySQL) 에 Full-text query 실행
2. Full-text Query 를 지원하는 다른 NoSQL 도입

편의상, 아래의 상황을 가정한다.

- 사용자는 서비스를 사용하기 위해 사용자의 이름을 등록해야한다.
- 사용자는 추가로 자신의 닉네임을 등록할 수 있다.
- 사용자는 다른 사람의 이름이나, 닉네임을 검색할 수 있다. 이때, 전체가 아닌 부분을 이용해 검색을 하더라도, 검색이 되어야 한다. 검색어는 2글자 이상이어야 한다.

예를 들어, 이름 : Neophocae**na p**hocaenoides, 닉네임 : 상괭이 이라고 하였을 때, 아래의 검색어 모두 허용한다.

- Neophocaena phocaenoides
- nap
- neo
- 괭이
- 상괭

### MySQL 을 사용하지 않는 간단한 이유

어떠한 Index 도 생성하지 않은 테이블에서 검색을 한다면 다음과 같은 sql 을 사용할 것이다.
```sql
SELECT * FROM name_table n
WHERE n.name LIKE '%괭이%' OR n.nick_name LIKE '%괭이%'
```

MySQL 에서는 텍스트 검색을 돕기 위해 `Full-text Index`를 지원한다만 주로 사용하지는 않는다. MySQL 에서는 인덱스를 추가한 필드에 데이터가 추가될 때 마다 새로 인덱싱을 하며, Full-Text Index 도 예외는 아니다. 차이점이라 하면, Full-Text Index는 추가된 데이터만 인덱싱을 하지 않는다. Mysql(InnoDB, 5.6~) 의 built-in parser 는 ngram 인데, 이는 데이터를 추가하면 ngram_token_size 에 맞는 길이의 다른 토큰들도 생성하고, 이들도 인덱싱을 한다.

이러한 동작 때문에, DDL DML 을 실행시 블로킹되는 현상이 나타나기도 하고, [데드락](https://stackoverflow.com/questions/73388434/deadlock-because-of-fulltext-index-fts-doc-id-index-in-mysql) 이 발생한다는 후기도 들려온다.

좋게봐줘서 search query 만 오래걸린다면 사실 slave DB 만을 검색하도록 하는 등의 방식도 있겠지만, DML 에서 블로킹 혹은 데드락이 발생하는 기술을 도입하기엔 무리가 있다.