## 시퀀스 다이어그램

> 1. 회차 구매

```mermaid
sequenceDiagram
    autonumber
    actor User as 사용자
    participant API as Purchase Controller
    participant Facade as PurchaseFacade

    box rgb(240, 248, 255) User Domain
        participant USvc as UserDomainService
        participant URepo as UserRepository
    end

    box rgb(255, 245, 255) Episode Domain
        participant ESvc as EpisodeDomainService
        participant ERepo as EpisodeRepository
    end

    box rgb(245, 255, 245) Purchase Domain
        participant PSvc as PurchaseDomainService
        participant PRepo as PurchaseRepository
    end

    box rgb(255, 250, 240) Access Domain
        participant ASvc as AccessDomainService
        participant ARepo as AccessRepository
    end

    User->>API: 회차 구매 요청 (userId, episodeId, idempotencyKey)
    API->>API: 필수 요청값 검증 (누락 시 400)
    API->>Facade: purchaseEpisode(userId, episodeId, idempotencyKey)

    Note over Facade, ARepo: @Transactional 시작

    rect rgb(230, 240, 255)
        Note right of Facade: 1. 비관적 락 획득 & 사용자 검증
        Facade->>USvc: getUserForUpdate(userId)
        USvc->>URepo: SELECT * FROM users WHERE id = ? FOR UPDATE
        URepo-->>USvc: 결과 반환
        alt 존재하지 않는 사용자
            USvc-->>Facade: Throw UserNotFoundException
            Facade-->>API: Exception 전파 (404)
        else OK
            USvc-->>Facade: User Entity 반환 (Lock 획득 상태)
        end
    end

    rect rgb(240, 240, 240)
        Note right of Facade: 2. 회차 검증 및 정보 조회
        Facade->>ESvc: getEpisode(episodeId)
        ESvc->>ERepo: SELECT * FROM episode WHERE id = ?
        ERepo-->>ESvc: 결과 반환
        alt 존재하지 않는 회차
            ESvc-->>Facade: Throw EpisodeNotFoundException
            Facade-->>API: Exception 전파 (404)
        else OK
            ESvc-->>Facade: Episode(price 포함) 반환
        end
    end

    rect rgb(255, 240, 240)
        Note right of Facade: 3. 멱등성 및 중복 구매 확인
        Facade->>PSvc: checkIdempotencyAndPurchase(userId, episodeId, idempotencyKey)
        PSvc->>PRepo: idempotency_key 또는 (user_id, episode_id) 조회
        PRepo-->>PSvc: 결과 반환
        alt 이미 처리된 요청 또는 구매함
            PSvc-->>Facade: Throw AlreadyProcessedException
            Facade-->>API: Exception 전파 (409)
        else 미구매
            PSvc-->>Facade: OK
        end
    end

    rect rgb(230, 250, 230)
        Note right of Facade: 4. 핵심 비즈니스 로직 실행

        Facade->>USvc: validateCoin(user, episodeCost)
        alt 코인 부족 또는 잘못된 코인 형태 (0원, 음수)
            USvc-->>Facade: Throw InvalidCoinException
            Facade-->>API: Exception 전파 (400)
        else 잔액 충분
            USvc-->>Facade: OK
        end

        Facade->>USvc: deductCoin(user, episodeCost)
        Note right of USvc: 이미 Lock을 획득한 상태이므로 즉시 수정 가능
        USvc->>URepo: UPDATE users SET coin = coin - ? WHERE id = ?
        URepo-->>USvc: OK
        USvc-->>Facade: OK

        Facade->>PSvc: createPurchase(userId, episodeId, price, idempotencyKey)
        PSvc->>PRepo: INSERT INTO purchase
        alt DB Unique Key 충돌
            PRepo-->>PSvc: Throw DataIntegrityViolationException
            PSvc-->>Facade: Throw AlreadyPurchasedException
            Facade-->>API: Exception 전파 (409)
        else 성공
            PRepo-->>PSvc: OK
        end

        Facade->>ASvc: grantAccess(userId, episodeId)
        ASvc->>ARepo: INSERT INTO access
        Note right of ARepo: Unique Index로 최종 중복 방어
        ARepo-->>ASvc: OK
    end

    Note over Facade, ARepo: @Transactional 커밋 (Lock 자동 해제)

    Facade-->>API: 구매 완료 응답 (purchaseId, episode, deductedCoin, purchasedAt)
    API-->>User: 200 OK
```

> 2. 회차 열람

```mermaid
sequenceDiagram
    autonumber
    actor User as 사용자
    participant API as Access Controller
    participant Facade as AccessFacade

    box rgb(240, 248, 255) User Domain
        participant USvc as UserDomainService
        participant URepo as UserRepository
    end

    box rgb(255, 245, 255) Episode Domain
        participant ESvc as EpisodeDomainService
        participant ERepo as EpisodeRepository
    end

    box rgb(255, 250, 240) Access Domain
        participant ASvc as AccessDomainService
        participant ARepo as AccessRepository
    end

    User->>API: 회차 열람 요청 (userId, episodeId)
    API->>Facade: getEpisodeContent(userId, episodeId)

    Note over Facade, ARepo: @Transactional(readOnly = true)

    rect rgb(240, 240, 240)
        Note right of Facade: 1. 유효성 검증 (오케스트레이션)
        Facade->>USvc: validateUser(userId)
        USvc->>URepo: SELECT * FROM users WHERE id = ?
        URepo-->>USvc: 결과 반환
        alt 존재하지 않는 사용자
            USvc-->>Facade: Throw UserNotFoundException
            Facade-->>API: Exception 전파 (404)
        else OK
            USvc-->>Facade: OK
        end

        Facade->>ESvc: validateEpisode(episodeId)
        ESvc->>ERepo: SELECT * FROM episode WHERE id = ?
        ERepo-->>ESvc: 결과 반환
        alt 존재하지 않는 회차
            ESvc-->>Facade: Throw EpisodeNotFoundException
            Facade-->>API: Exception 전파 (404)
        else OK
            ESvc-->>Facade: OK
        end
    end

    rect rgb(220, 235, 255)
        Note right of Facade: 2. 열람 권한 확인
        Facade->>ASvc: checkAccess(userId, episodeId)
        ASvc->>ARepo: SELECT * FROM access WHERE user_id = ? AND episode_id = ?
        ARepo-->>ASvc: 결과 반환
        alt 권한 없음
            ASvc-->>Facade: Throw AccessDeniedException
            Facade-->>API: Exception 전파 (403)
        else 권한 존재함
            ASvc-->>Facade: OK (AccessDTO, grantedAt 포함)
        end
    end

    rect rgb(230, 250, 230)
        Note right of Facade: 3. 콘텐츠 조회
        Facade->>ESvc: getEpisodeContentDetail(episodeId)
        ESvc->>ERepo: SELECT * FROM episode WHERE id = ?
        ERepo-->>ESvc: Content 데이터 반환
        ESvc-->>Facade: OK (ContentDTO)

        Facade->>Facade: 응답 객체 조합 (episode + contentUrl + grantedAt)
    end

    Facade-->>API: Content 정보 반환 (episode, contentUrl, grantedAt)
    API-->>User: 200 OK
```