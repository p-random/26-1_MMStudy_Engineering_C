![](https://velog.velcdn.com/images/jaewon77/post/d7c18aaf-1e82-42ef-8469-0a5c4d39202c/image.png)

데이터를 저장하는 방식 중 하나로 하둡생태계에서 압축과 컬럼 기반 데이터 표현의 이점을 만들기 위해 개발한 파일 포맷이다. 

공식문서에는 다음과 같이 나와있다.

- 언어를 가리지 않습니다.
- 컬럼 기반 형식 - 파일이 행이 아니라 열로 구성되어, 스토리지 공간이 절약되고 분석 쿼리 속도가 향상됩니다.
- 분석(OLAP) 사용 사례, 그중에서도 기존의 OLTP 데이터베이스와 함께 사용하는 사용 사례에 사용됩니다.
- 데이터 압축과 해제의 효율이 매우 높습니다.
- 복잡한 데이터 유형과 고급 중첩 데이터 구조를 지원![](https://velog.velcdn.com/images/jaewon77/post/4a5c2a42-e566-4be4-8557-64d733d70d69/image.png)

## Parquet 용어 사전 

| row | user_id | ts(시간)              | country | price |
| --: | ------: | ------------------- | ------- | ----: |
|   1 |     101 | 2026-01-01 09:00:01 | KR      |  1200 |
|   2 |     101 | 2026-01-01 09:00:10 | KR      |   500 |
|   3 |     102 | 2026-01-01 09:02:00 | JP      |   300 |
|   4 |     103 | 2026-01-01 10:00:00 | KR      |   800 |
|   5 |     104 | 2026-01-01 10:05:00 | US      |  1500 |
|   6 |     105 | 2026-01-01 10:06:00 | KR      |   700 |
|   7 |     106 | 2026-01-01 11:00:00 | KR      |   200 |
|   8 |     107 | 2026-01-01 11:01:00 | JP      |   900 |
|   9 |     108 | 2026-01-01 11:02:00 | KR      |   400 |
|  10 |     109 | 2026-01-01 11:03:00 | US      |  1100 |

위와 같은 데이터 예시가 있다고 가정해보자 


- Block(hdfs block) 
예시 데이터는 작아서 clicks.parquet가 블록 1개 안에 다 들어간다고 치자.
 
- File (파일 구조 + 메타데이터)

(앞쪽) 실제 데이터(각 row group의 column chunk들)

(뒤쪽) File Metadata (스키마, row group 위치/크기, 각 컬럼 통계 등)

(맨끝) footer 위치를 가리키는 magic bytes

 
- Row group

데이터를 논리적으로 행으로 수평분할한 단위.

Row Group 0: row 1~5

Row Group 1: row 6~10

row group에 대해 물리적인 구조로 나뉘어졌다고 보장할 수 없다. row group은 dataset의 각 column에 대한 column chunk로 구성된다.

 

- Column chunk

Row Group 0의 Column Chunks

user_id chunk: [101, 101, 102, 103, 104]

ts chunk: [09:00:01, 09:00:10, 09:02:00, 10:00:00, 10:05:00]

country chunk: [KR, KR, JP, KR, US]

price chunk: [1200, 500, 300, 800, 1500]

 

- Page

Column chunk는 page로 나뉜다. page는 압축 및 인코딩 측면에서 개념적으로 분할할 수 없는 단위다. interleaved 컬럼 청크에 여러 페이지 타입이 있을 수 있다.

Page 0 (country): [KR, KR, JP]

Page 1 (country): [KR, US]

Interleave: 읽고 쓸 때 성능을 높히기 위해 데이터가 서로 인접하지 않도록 배열하는 방법

```
HDFS
└── Block #0 (128MB)
    └── clicks.parquet  (Parquet File)
        ├── Row Group 0  (rows 1~5)
        │   ├── Column Chunk: user_id
        │   │   └── Pages: [page0 ...]
        │   ├── Column Chunk: ts
        │   │   └── Pages: [page0 ...]
        │   ├── Column Chunk: country
        │   │   ├── Page0: [KR, KR, JP]
        │   │   └── Page1: [KR, US]
        │   └── Column Chunk: price
        │       └── Pages: [page0 ...]
        ├── Row Group 1  (rows 6~10)
        │   └── ... (동일 구조)
        └── File Metadata (footer)

```


## 병렬화 단위

MapReduce - File/Row Group
SUM, AVG, COUNT와 같이 전체 데이터를 집계하는 연산을 하는 경우 특정 칼럼만 집계할 수 있어서 성능이 좋다.

IO - Column chunk
필요한 칼럼만 읽고 쓸 수 있기 때문에 불필요한 I/O를 줄일 수 있다.

Encoding/Compression - Page
같은 칼럼 안에 있는 데이터끼리는 비슷한 특성을 갖는다. 이 특성을 이용해 각 칼럼마다 최적의 압축과 인코딩 방식을 선택하면 공간을 절약할 수 있다.


![](https://velog.velcdn.com/images/jaewon77/post/886f4150-c587-4a7e-a2d7-836a8e66e273/image.png)

같은 컬럼에는 종종 유사한 데이터가 나열된다. 

이때 필요한 데이터만 디스크로부터 읽어 I/O를 최소화하고, 데이터 크기를 줄일수 있다. 

특히 같은 문자열의 반복은 매우 작게 압축할 수 있다. 데이터의 종류에 따라 다르지만, 열 지향 데이터베이스는 압축되지 않은 행 지향 데이터 베이스와 비교하면 1/10 이하로 압축 가능하다. 

# Parquet 의 효율성 

## 열 단위 저장

- row 기반: colA, colB, colC...가 한 줄에 같이 붙어 있으니

 colA만 필요해도 각 row 전체를 읽는 I/O가 발생하기 쉬움.

- Parquet: colA 블록, colB 블록이 분리돼 있어서

 쿼리가 SELECT colA면 colA에 해당하는 바이트 범위만 읽어오면 됨.
 
## Row Group 단위 스킵 

Row Group (큰 덩어리, 예: 128MB 단위)

각 Row Group 안에

Column Chunk (컬럼별 덩어리)

그 안이 다시

Page (더 작은 단위)

인 것을 위에서 확인했다.

![](https://velog.velcdn.com/images/jaewon77/post/aec0f8c9-e439-4a97-9842-78454bcf1e68/image.png)

핵심은 Row Group마다 통계(metadata) 가 들어간다는 것. (colX의 min/max, null_count, value_count 같은 통계)


```
SELECT user_id, price
FROM clicks
WHERE country = 'KR'


```
country 컬럼 통계(ROW GROUP의 min/max, dictionary 등)로
KR이 없는 row group은 통째로 스킵 (가능할 때)

남은 row group에서 user_id, price 컬럼 chunk만 읽음
(ts 컬럼 chunk는 아예 I/O 안 함)

필요하면 페이지 단위로도 조금 더 스킵 


### 1. Dictionary Encoding

```
dict = {
    0: "Seoul"
    1: "New York",
    2: "Paris",
}

# ["Seoul", "Seoul", "Seoul", "New York", "Paris", "New York"]
data = [0,0,0,1,2,1]
```
자주 사용되는 값들의 Dictionary를 만들어놓고 해당 값을 참조하여 사용한다.

반복되는 문자열 데이터가 많은 경우에 적합하다.

### 2. Run-Length Encoding(RLE)

```
# [10,10,20,20,20,30]
data = [(10,2),(20,3),(30,1)]
```
값과 그 값이 반복된 횟수를 저장한다.

연속적으로 반복되는 데이터가 많은 경우 적합하다.

### 3. DELTA Encoding

```
# [100,101,102,99,103]
data = [100,1,1,-3,4]
```
데이터의 변화량을 저장한다.

시계열 데이터에 적합.

