# redisCache
redisCache processes
[B팀_ 김민식, 박건영, 박태오, 오인준, 한정연.pdf](https://github.com/user-attachments/files/24049540/B._.pdf)

프로젝트 개선점 (Redis_Cache)
1) TestDataController의 GetMapping 제거 또는 RequestMapping 사용
✅ 기존 코드
@PostMapping("/generate-data")
@GetMapping("/generate-data")

✅ 개선 코드 (GetMapping 제거)
@PostMapping("/generate-data")
또는 (RequestMapping 사용)
@RequestMapping("/generate-data")

✅ 개선 이유
•  데이터 생성 기능은 POST가 REST 규칙에 맞고 안전하다.
•  GetMapping을 제거하면 RESTful 구조에 부합한다.
•  RequestMapping 사용 시 중복이 줄어 유지보수가 용이하다

2) SearchService의 getPopularKeywords() @Cacheable 삭제 또는 getPopularKeywordsRaw() 사용
✅ 기존 코드
@Cacheable(value = "search", key = "'popular_keywords'")
public List<String> getPopularKeywords(int limit) {
    try {
        Set<String> keywords = stringRedisTemplate.opsForZSet().reverseRange(POPULAR_KEYWORDS_KEY, 0, limit - 1);
        if (keywords == null) return List.of();
        return new ArrayList<>(keywords);
    } catch (RuntimeException ex) {
        safePurgeCorrupted();
        return List.of();
    }
}

✅ 개선 코드 
public List<String>  getPopularKeywords (int limit) {
    try {
        Set<String> keywords = stringRedisTemplate.opsForZSet().reverseRange(POPULAR_KEYWORDS_KEY, 0, limit - 1);
        if (keywords == null) return List.of();
        return new ArrayList<>(keywords);
    } catch (RuntimeException ex) {
        safePurgeCorrupted();
        return List.of();
    }
}

✅ 개선 이유
•  실시간 검색어는 초단위로 변하기 때문에 애플리케이션 캐시(@Cacheable) 사용 시 최신 데이터가 반영되지 않을 수 있다.
•  Redis는 메모리 기반 저장소로 실시간 조회에 최적이며, 실시간 인기 검색어 기능의 표준 방식으로 사용된다.
•  따라서 캐시 어노테이션을 제거하고 Redis 직접 조회 방식이 적합하다.

3) script 파일 search(btn) 수정
(addUserSearchKeyword / updatePopularKeywords 제거 후 loadKeywords 통합)

✅ 기존 코드
addUserSearchKeyword(keyword);
await updatePopularKeywords();

✅ 개선 코드
loadKeywords();
//    addUserSearchKeyword(keyword);
//    await updatePopularKeywords();

✅ 개선 이유
•  JS 또는 화면에 저장하는 방식은 간단하지만 다음 문제점이 있다:
•	사용자별 저장 불가
•	기기 간 동기화 불가
•	데이터 쉽게 소실
•  Redis에 저장하면 다음 장점이 있다:
•	사용자별 기록 유지 가능
•	모든 기기에서 동일한 기록 표시
•	검색 분석, 추천 기능 확장 가능
•  또한 동일한 저장 구조 사용으로 코드 일관성과 유지보수가 좋아진다.
•  RequestMapping 사용 시 중복이 줄어 유지보수가 용이하다
4) SearchService의 processSearch() 메서드의 Redis 호출 합치기
✅ 기존 코드
@CacheEvict(cacheNames = "search", allEntries = true)
    public void processSearch(String keyword) {
        saveOrUpdateSearchKeyword(keyword);
        updateRealTimeRanking(keyword);
        updateRecentKeywords(keyword);
    }

private void saveOrUpdateSearchKeyword(String keyword) {
        SearchKeyword searchKeyword = searchKeywordRepository
                .findByKeyword(keyword)
                .orElse(SearchKeyword.builder()
                        .keyword(keyword)
                        .searchCount(0L)
                        .build());
        searchKeyword.incrementSearchCount();
        searchKeywordRepository.save(searchKeyword);
    }

private void updateRealTimeRanking(String keyword) {
        stringRedisTemplate.opsForZSet().incrementScore(POPULAR_KEYWORDS_KEY, keyword, 1);
    }

private void updateRecentKeywords(String keyword) {
        stringRedisTemplate.opsForList().remove(RECENT_KEYWORDS_KEY, 0, keyword);
        stringRedisTemplate.opsForList().leftPush(RECENT_KEYWORDS_KEY, keyword);
        stringRedisTemplate.opsForList().trim(RECENT_KEYWORDS_KEY, 0, 9);
    }


✅ 개선 코드
@CacheEvict(cacheNames = "search", allEntries = true)
    public void processSearch(String keyword) {
        saveOrUpdateSearchKeyword(keyword);
        updateRedisForSearch(keyword);
    }

private void saveOrUpdateSearchKeyword(String keyword) {
        SearchKeyword searchKeyword = searchKeywordRepository
                .findByKeyword(keyword)
                .orElse(SearchKeyword.builder()
                        .keyword(keyword)
                        .searchCount(0L)
                        .build());
        searchKeyword.incrementSearchCount();
        searchKeywordRepository.save(searchKeyword);
    }


    private void updateRedisForSearch(String keyword) {
        stringRedisTemplate.executePipelined(new SessionCallback<Object>() {
            @Override
            public Object execute(RedisOperations operations) throws DataAccessException {
                ZSetOperations<String, String> zOps = operations.opsForZSet();
                ListOperations<String, String> lOps = operations.opsForList();

                zOps.incrementScore(POPULAR_KEYWORDS_KEY, keyword, 1.0);

                lOps.remove(RECENT_KEYWORDS_KEY, 0, keyword);
                lOps.leftPush(RECENT_KEYWORDS_KEY, keyword);
                lOps.trim(RECENT_KEYWORDS_KEY, 0, 9);

                return null;
            }
        });
    }

✅ 개선 이유
기존 코드에서는 updateRealTimeRanking / updateRecentKeywords 안에서 각각 독립적으로 Redis 명령을 보내 ZINCRBY + LREM + LPUSH + LTRIM까지 총 4번의 왕복이 생기고 있었다.
이를 파이프라인으로 연결하여 1번의 왕복으로 Redis 명령을 처리할 수 있도록 수정하였다.

5) SearchService의 getPopularKeywords() @Cacheable 삭제 또는 getPopularKeywordsRaw() 사용
✅ 기존 코드
private void updateRedisBulkOnly(Map<String, Long> increments, List<String> recent) {
        stringRedisTemplate.executePipelined((RedisConnection conn) -> {
            var ser = stringRedisTemplate.getStringSerializer();
            byte[] zkey = ser.serialize(POPULAR_KEYWORDS_KEY);
            byte[] lkey = ser.serialize(RECENT_KEYWORDS_KEY);

            for (Map.Entry<String, Long> e : increments.entrySet()) {
                conn.zIncrBy(zkey, e.getValue(), ser.serialize(e.getKey()));
            }
            if (recent != null && !recent.isEmpty()) {
                for (String kw : recent) {
                    conn.lRem(lkey, 0, ser.serialize(kw));
                    conn.lPush(lkey, ser.serialize(kw));
                }
                conn.lTrim(lkey, 0, 9);
            }
            return null;
        });
    }

✅ 개선 코드 
private void updateRedisBulkOnly(Map<String, Long> increments, List<String> recent) {
        stringRedisTemplate.executePipelined(new SessionCallback<Object>() {
            @Override
            public Object execute(RedisOperations ops) {
                ZSetOperations<String, String> zOps = ops.opsForZSet();
                ListOperations<String, String> lOps = ops.opsForList();

                increments.forEach((keyword, delta) ->
                        zOps.incrementScore(POPULAR_KEYWORDS_KEY, keyword, delta)
                );

                if (recent != null && !recent.isEmpty()) {
                    for (String kw : recent) {
                        lOps.remove(RECENT_KEYWORDS_KEY, 0, kw);
                        lOps.leftPush(RECENT_KEYWORDS_KEY, kw);
                    }
                    lOps.trim(RECENT_KEYWORDS_KEY, 0, 9);
                }

                return null;
            }
        });
    }

✅ 개선 이유
RedisConnection 방식은 Redis 명령을 직접 제어할 수 있어 학습용으로는 좋지만, 직렬화를 직접 처리해야 하고 코드가 복잡해져 유지보수성이 크게 떨어진다. 
반면 RedisTemplate의 고수준 API(opsForZSet, opsForList 등)를 파이프라인과 함께 사용하면 직렬화가 자동으로 처리된다. 또한 명령이 무엇을 처리하는지 코드만 보고도 쉽게 이해할 수 있어 가독성이 높아진다.
이는 스프링이 의도한 방식(고수준 API의 사용)이라 팀 개발 환경에서 일관성이 유지되고 확장성, 안전성 측면에서도 훨씬 유리하다. 성능은 두 방식이 동일하기 때문에 성능을 이유로 RedisConnection을 쓸 필요도 없다. 결국 실제 프로젝트나 협업에서는 고수준 API + 파이프라인 방식이 더 안정적이고 실용적이다.

6) clearAllCacheFast()에 캐시삭제 어노테이션 추가(@CacheEvict)
- (같은 이유로 fastGenerateAndSnapshot()에 (@CacheEvict추가))

✅ 기존 코드
public void clearAllCacheFast() {
    stringRedisTemplate.delete(POPULAR_KEYWORDS_KEY);
    stringRedisTemplate.delete(RECENT_KEYWORDS_KEY);
}

✅ 개선 코드 
clearAllCacheFast()에 어노테이션추가
@CacheEvict(cacheNames = "search", allEntries = true)
public void clearAllCacheFast() {
    stringRedisTemplate.delete(POPULAR_KEYWORDS_KEY);
    stringRedisTemplate.delete(RECENT_KEYWORDS_KEY);
}

✅ 개선 이유
•  Redis 데이터와 스프링 캐시는 서로 다른 저장소이다.
•  Redis 데이터를 삭제해도 @Cacheable 캐시는 남아있을 수 있다.
•  이 경우 캐시가 Redis보다 오래된 데이터를 반환하는 문제가 발생한다.
•  @CacheEvict를 함께 사용하면 스프링 캐시도 즉시 제거할 수 있다.
•  따라서 Redis와 스프링 캐시의 데이터 일관성을 위해 두 곳을 모두 지워야 한다.


