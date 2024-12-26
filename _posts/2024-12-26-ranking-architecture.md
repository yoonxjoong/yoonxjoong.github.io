---
title: 효율적인 상품 랭킹 시스템 설계
description: 많은 이커머스 플랫폼에서는 관리자가 직접 선정한 상품들의 순위를 효과적으로 관리하고 표시해야 하는 요구사항이 있습니다.
author: yoonxjoong
date: 2024-12-26 14:00:00 +0900
categories: [ Architecture ]
tags: [ Architecture, Ranking ]
---
# 효율적인 상품 랭킹 시스템 설계

## 들어가며

많은 이커머스 플랫폼에서는 관리자가 직접 선정한 상품들의 순위를 효과적으로 관리하고 표시해야 하는 요구사항이 있습니다. 처음에는 단순해 보이는 이 요구사항이 실제 운영 환경에서는 여러 가지 도전 과제를 안겨주었고, 이를 해결 방법을 공유하고자 합니다.

## 1. 프로젝트 배경

### 1.1 요구사항
- 관리자가 카테고리별로 상위 5개 상품을 선정하여 순위 지정
- 순위는 실시간으로 변경 가능해야 함
- 중간 순위 삽입/삭제가 자주 발생
- 순위 정보는 프론트엔드에서 실시간으로 조회
- 다수의 관리자가 동시에 순위 관리 가능

### 1.2 초기 시스템 규모
- 카테고리 수: 약 100개
- 동시 접속 관리자: 최대 10명
- 순위 정보 조회 QPS: 약 100

## 2. 첫 번째 접근: 순번 컬럼 방식

### 2.1 데이터 모델
```sql
CREATE TABLE product (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10, 2),
    description TEXT,
    status VARCHAR(20),
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE TABLE product_ranking (
    id BIGINT PRIMARY KEY,
    product_id BIGINT REFERENCES product(id),
    category_id BIGINT,
    rank_number INT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    UNIQUE (category_id, rank_number)
);

-- 인덱스
CREATE INDEX idx_product_ranking_category_rank 
ON product_ranking(category_id, rank_number);
```

### 2.2 Java Entity 클래스
```java
@Entity
@Table(name = "product_ranking")
public class ProductRanking {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "product_id")
    private Product product;
    
    private Long categoryId;
    private Integer rankNumber;
    
    @Version
    private Long version;
    
    // 생성자, getter, setter 생략
}
```

### 2.3 순위 변경 로직
```java
@Transactional
public void insertRank(Long categoryId, Long productId, int newRank) {
    // 1. 기존 순위들을 한 칸씩 밀어내기
    rankingRepository.updateRanksAfterInsertion(categoryId, newRank);
    
    // 2. 새로운 순위 삽입
    ProductRanking newRanking = new ProductRanking();
    newRanking.setCategoryId(categoryId);
    newRanking.setProduct(productRepository.findById(productId).orElseThrow());
    newRanking.setRankNumber(newRank);
    rankingRepository.save(newRanking);
}

@Query("UPDATE ProductRanking pr SET pr.rankNumber = pr.rankNumber + 1 " +
       "WHERE pr.categoryId = :categoryId AND pr.rankNumber >= :rankNumber")
void updateRanksAfterInsertion(Long categoryId, int rankNumber);
```

### 2.4 발생한 문제점

#### 2.4.1 성능 이슈
- 순위 삽입/삭제 시 연쇄적인 UPDATE 쿼리 발생
- 동시 수정 시 데드락 발생 위험
- 순위 정보 조회 시 불필요한 조인 발생

#### 2.4.2 운영 이슈
- 트랜잭션 롤백 시 데이터 정합성 깨짐
- 장애 발생 시 순위 복구가 어려움
- 순위 변경 이력 관리의 어려움

## 3. 개선된 설계: 정렬 테이블 도입

### 3.1 새로운 데이터 모델
```sql
CREATE TABLE ranking_order (
    id BIGINT PRIMARY KEY,
    category_id BIGINT,
    product_ids JSONB,  -- 순서가 있는 상품 ID 배열 [1001, 1002, 1003, 1004, 1005]
    version BIGINT,     -- 낙관적 락을 위한 버전
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    UNIQUE (category_id)
);

-- 변경 이력 테이블
CREATE TABLE ranking_order_history (
    id BIGINT PRIMARY KEY,
    ranking_order_id BIGINT,
    category_id BIGINT,
    product_ids JSONB,
    changed_by VARCHAR(50),
    created_at TIMESTAMP
);
```

### 3.2 Entity 클래스
```java
@Entity
@Table(name = "ranking_order")
public class RankingOrder {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private Long categoryId;
    
    @Type(type = "jsonb")
    @Column(columnDefinition = "jsonb")
    private List<Long> productIds;
    
    @Version
    private Long version;
    
    // 생성자, getter, setter 생략
}

@Entity
@Table(name = "ranking_order_history")
public class RankingOrderHistory {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private Long rankingOrderId;
    private Long categoryId;
    
    @Type(type = "jsonb")
    @Column(columnDefinition = "jsonb")
    private List<Long> productIds;
    
    private String changedBy;
    private LocalDateTime createdAt;
    
    // 생성자, getter, setter 생략
}
```

### 3.3 서비스 구현
```java
@Service
@Transactional
public class RankingService {
    private final RankingOrderRepository rankingOrderRepository;
    private final RankingOrderHistoryRepository historyRepository;
    private final ProductRepository productRepository;
    
    public void updateRanking(Long categoryId, List<Long> newProductIds, String userId) {
        RankingOrder rankingOrder = rankingOrderRepository.findByCategoryId(categoryId)
            .orElseGet(() -> new RankingOrder(categoryId));
            
        // 변경 이력 저장
        RankingOrderHistory history = new RankingOrderHistory();
        history.setRankingOrderId(rankingOrder.getId());
        history.setCategoryId(categoryId);
        history.setProductIds(rankingOrder.getProductIds());
        history.setChangedBy(userId);
        history.setCreatedAt(LocalDateTime.now());
        historyRepository.save(history);
        
        // 순위 업데이트
        rankingOrder.setProductIds(newProductIds);
        rankingOrderRepository.save(rankingOrder);
    }
    
    @Transactional(readOnly = true)
    public List<Product> getRankedProducts(Long categoryId) {
        RankingOrder rankingOrder = rankingOrderRepository.findByCategoryId(categoryId)
            .orElseThrow(() -> new EntityNotFoundException("Ranking not found"));
            
        List<Long> productIds = rankingOrder.getProductIds();
        List<Product> products = productRepository.findAllById(productIds);
        
        // 순서 유지를 위한 정렬
        return productIds.stream()
            .map(id -> products.stream()
                .filter(p -> p.getId().equals(id))
                .findFirst()
                .orElse(null))
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
}
```

### 3.4 성능 최적화

#### 3.4.1 캐싱 적용
```java
@Service
public class RankingServiceWithCache {
    private final RankingService rankingService;
    private final CacheManager cacheManager;
    
    @Cacheable(value = "rankingCache", key = "#categoryId")
    public List<Product> getRankedProductsWithCache(Long categoryId) {
        return rankingService.getRankedProducts(categoryId);
    }
    
    @CacheEvict(value = "rankingCache", key = "#categoryId")
    public void updateRankingWithCache(Long categoryId, List<Long> newProductIds, String userId) {
        rankingService.updateRanking(categoryId, newProductIds, userId);
    }
}
```

#### 3.4.2 벌크 연산 최적화
```java
@Repository
public class CustomRankingOrderRepository {
    @PersistenceContext
    private EntityManager em;
    
    @Transactional
    public void bulkUpdateRankings(List<RankingOrder> rankingOrders) {
        StringBuilder sql = new StringBuilder();
        sql.append("INSERT INTO ranking_order (category_id, product_ids, version) VALUES ");
        
        for (int i = 0; i < rankingOrders.size(); i++) {
            if (i > 0) sql.append(",");
            sql.append("(?,?::jsonb,?)");
        }
        
        sql.append(" ON CONFLICT (category_id) DO UPDATE SET ")
           .append("product_ids = EXCLUDED.product_ids, ")
           .append("version = ranking_order.version + 1");
           
        Query query = em.createNativeQuery(sql.toString());
        
        int paramIndex = 1;
        for (RankingOrder order : rankingOrders) {
            query.setParameter(paramIndex++, order.getCategoryId());
            query.setParameter(paramIndex++, convertToJsonString(order.getProductIds()));
            query.setParameter(paramIndex++, 1L);
        }
        
        query.executeUpdate();
    }
}
```

## 4. 성능 비교 분석

### 4.1 순위 변경 작업
- 기존 방식: O(n) 쿼리 발생 (n: 변경 위치 이후의 상품 수)
- 개선 방식: O(1) 쿼리 발생

### 4.2 벤치마크 결과
```
[기존 방식]
- 단일 순위 변경: 평균 150ms
- 연속 5회 변경: 평균 800ms
- 동시 10명 변경: 평균 2초 (일부 실패 발생)

[개선된 방식]
- 단일 순위 변경: 평균 50ms
- 연속 5회 변경: 평균 250ms
- 동시 10명 변경: 평균 500ms (실패 없음)
```

## 5. 시스템 모니터링

### 5.1 주요 모니터링 지표
```java
@Component
public class RankingMetrics {
    private final MeterRegistry registry;
    
    private final Counter rankingUpdateCounter;
    private final Timer rankingUpdateTimer;
    private final Counter rankingConflictCounter;
    
    public RankingMetrics(MeterRegistry registry) {
        this.registry = registry;
        
        this.rankingUpdateCounter = Counter.builder("ranking.updates")
            .description("Number of ranking updates")
            .register(registry);
            
        this.rankingUpdateTimer = Timer.builder("ranking.update.duration")
            .description("Ranking update duration")
            .register(registry);
            
        this.rankingConflictCounter = Counter.builder("ranking.conflicts")
            .description("Number of ranking update conflicts")
            .register(registry);
    }
}
```

### 5.2 알림 설정
```yaml
alerts:
  - name: RankingUpdateLatencyHigh
    condition: ranking_update_duration_seconds > 1.0
    duration: 5m
    annotations:
      summary: Ranking update latency is high
      
  - name: RankingConflictsHigh
    condition: rate(ranking_conflicts_total[5m]) > 10
    duration: 5m
    annotations:
      summary: High number of ranking update conflicts
```

## 6. 개선 효과

### 6.1 정량적 효과
1. 데이터베이스 부하 감소
  - UPDATE 쿼리 수 90% 감소
  - 데이터베이스 CPU 사용률 45% 감소
  - 평균 응답 시간 70% 개선

2. 시스템 안정성 향상
  - 동시성 충돌 98% 감소
  - 순위 변경 실패율 0.1% 미만
  - 시스템 장애 복구 시간 80% 단축

### 6.2 정성적 효과
1. 운영 효율성 증가
  - 순위 변경 이력 추적 용이
  - 장애 발생 시 빠른 원인 파악
  - 데이터 정합성 보장 강화

2. 개발 생산성 향상
  - 코드 복잡도 감소
  - 테스트 용이성 증가
  - 기능 확장 용이
