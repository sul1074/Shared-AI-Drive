# 코어 시스템 아키텍처 설계 의도 · 특징 및 성능 최적화 검증

## 0. 시스템 개요 및 아키텍처 흐름

**[시스템 아키텍처 다이어그램]**
```mermaid
flowchart TB
    %% 색상 및 스타일 정의
    classDef client fill:#f9f9f9,stroke:#333,stroke-width:2px;
    classDef api fill:#e1f5fe,stroke:#0288d1,stroke-width:2px;
    classDef aop fill:#fce4ec,stroke:#d81b60,stroke-width:2px,stroke-dasharray: 5 5;
    classDef facade fill:#e8f5e9,stroke:#388e3c,stroke-width:2px;
    classDef service fill:#fff3e0,stroke:#f57c00,stroke-width:2px;
    classDef worker fill:#ede7f6,stroke:#512da8,stroke-width:2px;
    classDef infra fill:#eceff1,stroke:#607d8b,stroke-width:2px;
    classDef external fill:#ffebee,stroke:#c62828,stroke-width:2px;

    Client([Client / Frontend]):::client

    subgraph SpringBoot ["Spring Boot Application Core"]
        direction TB

        subgraph Presentation ["Presentation Layer"]
            Controller["DriveController<br/>(REST API)"]:::api
            Aspect(("PermissionAspect<br/>(@RequirePermission)")):::aop
        end

        subgraph Facade ["Facade Layer"]
            direction LR
            QueryFacade["DriveQueryFacade<br/>(조회 전용 / Read-Only)"]:::facade
            CommandFacade["DriveCommandFacade<br/>(상태 변경 / Transactional)"]:::facade
        end

        subgraph DomainServices ["Business Service Layer"]
            DriveSvc["Drive Services<br/>(File, Folder, Manage, ...)"]:::service
            PermSvc["PermissionService<br/>(ReBAC 동적 권한 검증)"]:::service
            EmbedSvc["DriveEmbeddingService<br/>(RAG 통신 및 상태 동기화)"]:::service
        end

        subgraph Background ["Background Worker"]
            Scheduler{"DriveEmbeddingScheduler<br/>(1초 주기 Polling)"}:::worker
        end
    end

    subgraph Infrastructure ["Infrastructure & External"]
        direction LR
        RDB[("PostgreSQL")]:::infra
        Storage[("Local File Storage<br/>(On-Premise)")]:::infra
        RAG["RAG Server<br/>(FastAPI)"]:::external
    end

    %% 흐름 연결 (Client -> AOP Proxy -> Controller -> Facade)
    Client -->|HTTP Request| Aspect
    Aspect -->|"권한 검증 위임"| PermSvc
    Aspect ==>|"인가 성공 후 proceed()"| Controller
    Controller ==>|"조회 요청"| QueryFacade
    Controller ==>|"상태 변경 요청"| CommandFacade

    %% Facade -> Services
    QueryFacade --> DriveSvc
    CommandFacade --> DriveSvc
    CommandFacade -->|"다건 권한/경계 검증"| PermSvc
    CommandFacade -. "임베딩 상태 갱신 트리거" .-> EmbedSvc

    %% Background -> Services
    Scheduler -->|1. 타겟 선점 SKIP LOCKED| EmbedSvc

    %% Services -> Infrastructure
    DriveSvc -->|"Data Read/Write"| RDB
    DriveSvc -->|"File Upload/Download"| Storage
    PermSvc -->|"계층 권한 조회"| RDB
    
    EmbedSvc -->|"2. 파일 존재 검증"| Storage
    EmbedSvc <-->|"3. 벡터 임베딩 API 요청"| RAG
    EmbedSvc -->|"4. 상태 전파 (Bulk Update)"| RDB
```

**계층별 주요 역할**
- **Presentation Layer**: 클라이언트 요청을 수신하며, AOP(`@RequirePermission`)를 통해 비즈니스 로직 진입 전 단건 자원에 대한 권한을 검증합니다.

- **Facade Layer**: 여러 도메인 서비스(폴더, 파일, 권한 등)가 복합적으로 얽히는 작업의 트랜잭션 경계를 설정하고 오케스트레이션을 담당하여 서비스 간 순환 참조를 방지합니다.

- **Business Service Layer**: 드라이브 데이터 제어, 다형성 기반 권한 검증(ReBAC) 등 각 도메인의 핵심 비즈니스 로직을 처리합니다.

- **Background Worker**: RDB의 `SKIP LOCKED`를 활용해 대상을 선점하는 즉시 상태를 `PROCESSING`으로 변경하며 트랜잭션을 종료하고 벡터 임베딩을 동기로 요청합니다. 메인 스레드와는 분리된 워커 스레드들이 타겟을 한 건씩 조회하여 순차적으로 임베딩 파이프라인을 처리합니다.


<br>

## 1. 드라이브 시스템

### 1-1. 아키텍처 설계 및 트레이드오프
- **문제 인식:** 디렉토리 관리에 단일 부모-자식 ID 참조(Adjacency List) 방식만 사용할 경우, 직속 하위(1-Depth) 탐색에는 유리하지만 **하위/상위의 모든 계층 자원 (N-Depth)**을 탐색할 때 재귀 쿼리를 발생시켜 트리의 깊이가 깊어질수록 DB I/O 부하가 커집니다.

- **해결책 (경로 열거 패턴 도입):** 동일 폴더 내 이름 중복 방지와 같은 DB 제약 및 1-Depth 탐색을 위해 기존 인접 목록 방식(`parent_folder_id`)을 유지하면서, 다중 계층(N-Depth) 탐색 성능을 높이기 위해 `logicalPath`(예: `/uuidA/uuidB/uuidC/`) 컬럼을 추가했습니다. `logicalPath`에는 자기 자신의 UUID도 포함되며, 컬럼 길이와 운영 복잡도를 고려해 최대 깊이는 12로 제한했습니다.

- **조회 성능 향상** 
   - **하위 자원 전체 조회:** prefix 조건(`LIKE '/uuid/.../%'`)과 PostgreSQL의 `varchar_pattern_ops` 인덱스를 결합하여, 하위 폴더/파일을 재귀 쿼리 없이 단일 쿼리로 탐색할 수 있도록 했습니다.

   - **상위 자원 전체 조회:** 엔티티의 `getAncestorFolderIds()`를 통해 자기 자신을 포함한 모든 상위 폴더 ID를 logicalPath를 이용해 애플리케이션 메모리 단에서 파싱하고, IN 절을 활용해 1번의 쿼리로 모든 조상 폴더들을 조회할 수 있습니다.

- **트레이드오프 관리 (Read vs Write):** 폴더 이동(`moveFolders`) 시 하위 폴더들의 `logicalPath`를 함께 수정해야 하는 쓰기 비용이 발생합니다. 하지만 본 시스템에서는 **쓰기(수정)보다 조회(탐색) 빈도가 높고, 권한 검증 및 폴더 임베딩 상태 전파를 위해 상·하위 자원을 빈번하게 조회해야 한다는 요구사항**을 우선하여 이 트레이드오프를 선택했습니다.

### 1-2. 대용량 데이터 쓰기 성능 및 영속성 컨텍스트 정합성 관리
- **QueryDSL Bulk Update:** 폴더 이동(`moveItems`)이나 삭제(`deleteItems`) 등 하위 항목 전체에 대한 Update가 필요한 경우, JPA의 변경 감지가 아닌 QueryDSL 기반의 Bulk Update(`예: bulkUpdateDescendantsPathList`) 를 통해 성능 병목을 완화시켰습니다.

- **영속성 컨텍스트 관리:** Bulk 연산 수행 전 `em.flush()`로 보류된 변경을 DB에 반영하고, 수행 후 `em.clear()`로 1차 캐시를 비워 Bulk Update 이후 영속성 컨텍스트가 오래된 상태를 계속 참조하는 문제를 방지했습니다.

### 1-3. Facade 아키텍처를 통한 트랜잭션 및 의존성 관리
- **문제 인식:** 폴더 이동, 파일 삭제, 권한 검증 등은 `DriveFolderService`, `DriveFileService`, `DrivePermissionService` 등 다수의 도메인 서비스가 복합적으로 얽히는 작업입니다. 이를 단일 서비스에서 모두 처리하려 할 경우, 비즈니스 로직의 전체적인 흐름을 따라가기 어려워지고 코드의 가독성이 저하되어 유지보수와 확장이 어려운 구조가 될 우려가 컸습니다.

- **해결책 (계층 분리 및 CQS):** 비즈니스 로직(Service)과 흐름 제어(Facade) 계층을 분리했습니다. 또한 데이터의 상태를 변경하는 `DriveCommandFacade`와 조회를 담당하는 `DriveQueryFacade`로 나누어 명령과 조회의 책임(CQS)을 논리적으로 분리했습니다.

- **트랜잭션 오케스트레이션:** 다중 도메인 서비스의 조합이 필요한 작업은 Facade 계층의 메서드에서 트랜잭션 경계를 설정합니다. 여러 DB 작업을 하나의 논리적 작업 단위로 묶어 예외 발생 시 함께 롤백되도록 하여 DB 내부 작업의 원자성을 확보했습니다. 스프링 트랜잭션의 기본 전파 옵션(`Propagation.REQUIRED`)에 따라 하위 서비스가 별도 전파 옵션을 지정하지 않는 경우 Facade의 최상위 트랜잭션에 합류합니다.

- **단일 책임 원칙:** Facade는 흐름 제어와 트랜잭션 경계 설정만 담당하고, 실제 비즈니스 로직은 하위 서비스로 위임하여 객체 지향적인 유연성과 단위 테스트 용이성을 향상시켰습니다.

```text
[Presentation Layer]
DriveController (REST API)
 │
 ├── [Facade Layer : 트랜잭션 경계 & 비즈니스 오케스트레이션]
 │    │
 │    ├── DriveQueryFacade (조회 전용, @Transactional(readOnly = true))
 │    │    ├── DriveFolderService
 │    │    ├── DriveFileService
 │    │    ├── DriveRefService
 │    │    └── DriveManageService 
 │    │
 │    └── DriveCommandFacade (상태 변경 전용, @Transactional)
 │         ├── DrivePermissionService (권한 경계 및 유효성 검증)
 │         ├── DriveRefService (AI 문서 참조 토큰 발급 및 조회)
 │         ├── DriveManageService (드라이브 관리)
 │         ├── DriveFolderService (폴더 제어)
 │         ├── DriveFileService (파일 제어)
 │         └── DriveEmbeddingService (임베딩 상태 동기화)
 │
[Background Worker Layer : 스케줄링]
DriveEmbeddingScheduler (FOR UPDATE SKIP LOCKED 제어)
 └── DriveEmbeddingService (독립적인 트랜잭션으로 임베딩 처리)
```

### 1-4. DB와 물리 스토리지 간의 분산 환경 정합성 관리
- **커넥션 풀 고갈 및 이중 쓰기 방어:** 파일 업로드 같은 긴 네트워크 I/O 작업 시 DB 트랜잭션을 점유하면 커넥션 풀 고갈 장애가 발생할 수 있습니다. 이를 예방하기 위해 스토리지 업로드 로직과 DB 트랜잭션을 분리했습니다. 이로 인해 시스템 간의 원자성이 어긋나는 고아 파일 이슈를 제어해야 했습니다.

- **실행 순서 및 보상 작업:** 물리 스토리지에 먼저 업로드한 뒤 DB 트랜잭션을 수행하며, DB 저장이 실패하면 이미 업로드된 파일을 삭제하는 **보상 작업**을 수행하도록 했습니다. 이를 통해 고아 파일이 남을 가능성을 낮췄습니다. 다만 스토리지 삭제 자체도 실패할 수 있으므로, 이 방식이 DB와 파일 스토리지 사이의 완전한 분산 원자성을 보장하는 것은 아닙니다.

### 1-5. 최상위 루트 설계 및 API 진입점 분리
본 시스템의 디렉토리 구조는 최상위 루트를 물리적인 폴더로 생성하지 않고, 식별자를 `folderId = null`로 취급하는 가상 루트 방식을 채택했습니다. 더불어 조회 및 생성 시, 내 드라이브(`/my-drive`), 공유 드라이브(`/shares`), 일반 폴더(`/folders`)로 API 진입점을 분리했습니다. 이 아키텍처는 다음과 같은 설계적 의도와 트레이드오프를 갖습니다.

**가상 루트 및 API 분리를 채택한 설계적 의도**
- **도메인 책임 분리와 데이터 중복 방지:** 최상위 루트의 메타데이터(`이름, 소유자 등`)는 이미 드라이브 본연의 속성입니다. 이를 최상위 폴더로 한 번 더 생성하는 것은 데이터 중복이라 판단하여, 드라이브(`컨테이너`)와 폴더(`내부 자원`)의 도메인 역할을 분리했습니다.

- **구조적 조작 차단:** 최상위 루트가 물리적 폴더로 존재할 경우, 요청 입력으로 이를 삭제, 이동, 이름 변경하려 할 때 이를 막기 위한 방어 로직(`depth == 0`)이 서비스 레이어 곳곳에 강제됩니다. 가상 루트 방식은 폴더 조작 API의 대상 자체가 되지 않으므로 이러한 리스크를 사전에 차단할 수 있습니다.

**설계적 한계점**
- **클라이언트 라우팅 복잡성 증대 (다형성의 부재):** 모든 자원이 단일 폴더로 추상화되지 않기 때문에, 클라이언트는 브레드크럼(`상단 경로`)을 클릭할 때 단순히 `folderId`만으로 단일 API를 호출할 수 없습니다. 이를 해결하기 위해 백엔드 응답 모델에 `type` 필드(`MY_DRIVE`, `SHARED_DRIVE`, `DRIVE_FOLDER`)를 추가해야 했으며, 프론트엔드 역시 이 `type`에 따라 서로 다른 API 엔드포인트로 분기 라우팅을 처리해야 합니다.

- **객체지향적 처리의 한계:** 모든 계층이 동일한 폴더 속성을 가지지 않으므로, 트리 탐색이나 루트 검증을 수행할 때 서비스 레이어 곳곳에 `if (folderId == null)`과 같은 예외 분기문이 일부 잔존하게 되었습니다.

---
<br>

## 2. 벡터 임베딩 파이프라인과 폴더 임베딩 상태 동기화
파일이 업로드되면 AI가 읽을 수 있도록 벡터 임베딩하는 백그라운드 병렬 Worker와, 폴더 하위의 파일들의 임베딩 여부 상태에 따른 폴더 임베딩 상태를 동기화하는 로직입니다.

### 2-1. SELECT FOR UPDATE SKIP LOCKED 기반의 스케줄링
메모리 큐만 사용할 경우 서버 프로세스 종료 시 메모리 내 작업 상태를 잃을 수 있습니다. 이를 피하기 위해 DB 테이블을 지속성 있는 작업 대기열로 활용하고, `SELECT FOR UPDATE SKIP LOCKED`로 여러 워커가 동일 타겟을 동시에 선점하는 경쟁을 방지하면서 병렬 처리합니다.

**[임베딩 스케줄러 동작 시퀀스 다이어그램]**
```mermaid
sequenceDiagram
    autonumber

    participant W as Worker Thread<br/>(Scheduler)
    participant SVC as EmbeddingService
    participant DB as DB & Storage
    participant API as RAG API Server

    %% 1. 스케줄링 및 헬스 체크
    loop Every 1 Second
        W->>API: GET /rag/health (상태 및 버전 체크)
        API-->>W: UP & Version
        
        %% 2. 타겟 선점 및 상태 변경 (Transaction 1)
        Note over W, DB: Transaction 1: 동시성 제어 및 타겟 선점
        W->>SVC: 임베딩 타겟 조회 요청
        activate SVC
        SVC->>DB: findNextTarget() <br/>(Pessimistic Lock + SKIP LOCKED)
        DB-->>SVC: 우선순위 파일 엔티티 반환
        
        alt 유효한 파일 (스토리지 존재)
            SVC->>DB: 상태 업데이트 (PROCESSING)
        else 파일 유실
            SVC->>DB: 상태 업데이트 (MISSING)
        end
        SVC-->>W: 임베딩 작업(Task) 반환
        deactivate SVC

        %% 3. RAG API 호출 (Network I/O)
        Note over W, API: DB 트랜잭션 분리 후 I/O 블로킹 대기
        W->>API: POST /rag/upload-files (파일 벡터화 요청)
        API-->>W: RagApiResponse (성공/실패 및 토큰 사이즈)

        %% 4. 결과 반영 및 폴더 동기화 (Transaction 2)
        Note over W, DB: Transaction 2: 결과 반영 및 트리 동기화
        W->>SVC: 결과 저장 및 폴더 상태 동기화
        activate SVC
        
        SVC->>DB: 임베딩 결과 업데이트 (COMPLETED or FAILED)
        
        opt 상태가 COMPLETED인 경우
            Note right of SVC: 재귀 쿼리 없이 메모리에서<br/>자식->부모 상태 계산
            SVC->>DB: 관련 폴더 상태 Bulk Update (isAllEmbedded)
        end
        
        SVC-->>W: 전체 처리 완료
        deactivate SVC
    end
```

- **Worker Pool 기반 병렬 처리 및 부하 제어**
  - 외부 AI 모델(RAG 서버)로 동시에 전송하는 요청 수를 `FixedThreadPool`과 `AtomicInteger`를 이용해 `maxWorkers` 범위로 제한합니다. 이 제한은 **애플리케이션 인스턴스 단위**이므로, 다중 인스턴스 전체의 글로벌 동시 요청 수를 제한하려면 별도의 분산 세마포어나 외부 서버 측 제한이 필요합니다.
  - 메인 클라이언트 요청을 처리하는 Tomcat 스레드와 임베딩 워커 스레드를 분리하여, 임베딩 처리 지연이 일반 API 요청 스레드를 직접 점유하지 않도록 했습니다.

- **동시성 제어** 
  - 다중 인스턴스·다중 스레드 환경에서 동일한 파일이 중복 선점되는 것을 막기 위해 `SELECT FOR UPDATE SKIP LOCKED`를 사용합니다. 동일 행이 이미 잠겨 있으면 다른 워커는 해당 행을 기다리지 않고 다음 후보를 탐색합니다.
  - 타겟을 찾으면(선점하면) 상태를 `PROCESSING`으로 변경하고 트랜잭션을 커밋하여 DB 락을 해제합니다.
  - 이를 통해 DB 커넥션을 점유하는 시간을 최소화하며, 타 워커 스레드가 동일한 타겟에 접근하는 것을 막습니다.
  - 이후에 이루어지는 벡터 임베딩 API 요청은 DB 락과 트랜잭션이 해제된 상태에서 동기 작업으로 수행됩니다.

- **우선순위 기반 DB 폴링:** DB에서 6단계 우선순위에 따라 1건씩 타겟을 가져옵니다.
  1. **수동 요청 (PRIORITIZED):** 사용자가 명시적으로 최우선 처리를 요청한 파일
  2. **실패한 구버전 재처리 (FAILED 중 구버전):** 이전 모델에서 실패했던 파일의 우선 재시도
  3. **신규 업로드 파일 (READY):** 유저가 새로 업로드한 파일
  4. **완료된 구버전 마이그레이션 (COMPLETED 중 구버전):** 이미 서비스 중이나 새 버전 모델로 업데이트가 필요한 파일. **신규 파일(`READY`)의 처리를 지연시키지 않기 위해 우선순위를 후순위로 배치**했습니다.
  5. **좀비 태스크 복구 (PROCESSING - 30분 지연):** 인스턴스 다운 등의 이유로 `PROCESSING` 상태에서 멈춰서 방치된 태스크를 회수하여 재처리합니다.
  6. **일반 실패 재시도 (FAILED - 60분 경과):** OCR 모델의 처리 오류로 실패한 파일들의 주기적 재시도

- **물리적 존재 여부 검증:** 작업 할당 전 파일 스토리지(`fileStorageService`) 시스템 내 실제 존재 여부를 검증하여, 유실된 파일에 대해 무한 재시도를 하는 낭비를 막고 `MISSING` 상태로 격리합니다.

### 2-2. 폴더 임베딩 상태의 상향식 동기화

**[폴더 임베딩 상태 동기화 흐름]**

```mermaid
flowchart TD
    %% 스타일 정의
    classDef startEnd fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef db fill:#fce4ec,stroke:#c2185b,stroke-width:2px;
    classDef process fill:#e3f2fd,stroke:#1565c0,stroke-width:2px;
    classDef memory fill:#fff3e0,stroke:#f57c00,stroke-width:2px;
    classDef cond fill:#fff9c4,stroke:#fbc02d,stroke-width:2px;

    Start(["▶ 임베딩 상태 동기화 요청<br>(대상: 파일이 속한 폴더들)"]):::startEnd

    subgraph Step1 ["1. 필요 데이터 일괄 조회 (N+1 방지)"]
        direction TB
        FetchAncestors[("타겟 폴더 및 모든 조상 폴더 ID 수집")]:::db
        FetchFiles[("미완료 파일이 포함된 폴더 ID Set 조회")]:::db
        FetchChildren[("폴더별 미완료 자식 폴더 개수 Map 조회")]:::db
        
        FetchAncestors --> FetchFiles & FetchChildren
    end

    Start --> Step1

    Step2["2. Depth 역순 정렬<br>(Bottom-Up: 가장 깊은 자식 폴더부터 위로)"]:::process
    Step1 --> Step2

    subgraph Step3 ["3. 상태 평가 및 상향식 전파"]
        direction TB
        Loop{"정렬된 폴더 목록<br>순회 (For-each)"}:::cond
        
        CheckStatus["현재 폴더 상태 판별<br>(미완료 파일 없음 && 미완료 자식 폴더 0개)"]:::memory
        
        IsChanged{"기존 DB 상태와<br>변경점이 있는가?"}:::cond
        
        Collect["일괄 업데이트 목록(List)에<br>폴더 ID 추가 (Complete / Incomplete)"]:::memory
        
        Propagate["부모 폴더의 상태 Map 갱신<br>(자식이 완료됨 -> 부모의 미완료 카운트 -1)<br>(자식이 미완료됨 -> 부모의 미완료 카운트 +1)"]:::memory
        
        Loop -->|폴더 1건 꺼냄| CheckStatus
        CheckStatus --> IsChanged
        IsChanged -->|Yes| Collect
        Collect --> Propagate
        Propagate -->|다음 폴더로| Loop
        IsChanged -->|No 상태 유지| Loop
    end

    Step2 --> Step3

    subgraph Step4 ["4. 최종 DB 반영"]
        BulkUpdate[("수집된 ID 대상 Bulk Update<br>(완료된 폴더 일괄 True, 미완료 일괄 False)")]:::db
    end

    Step3 -->|모든 폴더 순회 완료| Step4
    Step4 --> End(["최종 완료"]):::startEnd
```

- **상태 전파의 복잡성:** 특정 하위 파일의 임베딩이 완료/실패 상태로 변경되면, 그 상태가 최상위 조상 폴더의 `isAllEmbedded` 상태까지 전파되어야 합니다 (`자식 -> 부모 -> 조상`).

- **상태 취합 및 일괄 업데이트:** 재귀적인 DB 쿼리 호출을 방지하기 위해, `updateFolderEmbStatusByFolderId` 내에서 타겟 폴더들의 트리 정보를 메모리에 올립니다. 이후 각 폴더의 미완료 자식 카운트(`incompleteChildCounts`)를 메모리상에서 계산하여 **Bottom-Up 방식으로 각 부모 폴더들의 최종 상태를 취합한 뒤, Bulk Update로 DB에 반영**합니다.

- 이 동기화 로직은 벡터 임베딩 스케줄러에 의한 임베딩뿐만 아니라 자원의 삭제, 이동, 임베딩 우선순위 변경 시에도 동일하게 호출되도록 되어있습니다.

---
<br>

## 3. ReBAC 기반 계층적 권한 제어 시스템
역할 기반(RBAC)과 함께, 자원 간의 계층적 관계를 해석하여 권한을 검증하는 관계 기반 접근 제어(ReBAC)를 구현했습니다.

### 3-1. 통합 권한 데이터 모델 설계

**[권한 테이블 ERD]**

```mermaid
erDiagram
    %% 논리적 다형성 관계 (물리적 FK 없음)
    TARGET_RESOURCES ||--o{ permission_tb : "Target (target_type, target_id)"
    SUBJECTS ||--o{ permission_tb : "Subject (subject_type, subject_id)"

    permission_tb {
        String id PK "UUID"
        TargetType target_type "Composite UK 구성, IDX (DRIVE, FOLDER, FILE...)"
        String target_id "Composite UK 구성, IDX"
        SubjectType subject_type "Composite UK 구성, IDX (USER, GROUP)"
        String subject_id "Composite UK 구성, IDX"
        AccessRoleType role_type "(VIEWER, EDITOR, OWNER, ADMIN)"
        String created_id "최초 생성자"
        LocalDateTime created_date "최초 생성일시"
        String modified_id "마지막 수정자"
        LocalDateTime modified_date "마지막 수정일시"
    }

    TARGET_RESOURCES {
        Domain Entity "신규 권한 대상 도메인"
    }

    SUBJECTS {
        Domain Entity "유저(User) 또는 그룹(Group)"
    }
```

도메인별로 권한 테이블(예: `project_permission`, `drive_permission` 등)을 분리하면 테이블이 너무 많아지고 권한 로직의 파편화가 발생합니다. 따라서 단일 `permission_tb` 테이블에서 공유가 가능한 모든 시스템 자원의 권한을 중앙 집중식으로 관리하는 단일 진실 공급원(`SSOT`)으로 설계했습니다. 

- **다형성 기반 확장:** `TargetType`과 `TargetId`의 조합으로 프로젝트, 프롬프트, 드라이브 등 신규 도메인을 권한 테이블 스키마 변경 없이 추가할 수 있도록 했습니다. 중복 권한 생성을 막기 위한 유일성은 `(target_type, target_id, subject_type, subject_id)` 복합 키 기준으로 관리합니다.

- **참조 무결성 트레이드오프:** 다형성 `target_id` 구조는 여러 도메인을 하나의 컬럼으로 참조하므로 일반적인 물리 FK를 걸기 어렵습니다. 따라서 대상 자원 삭제 시 권한 정리 등 참조 무결성의 일부를 애플리케이션 로직이 책임해야 하며, 이 부분을 단일 테이블 구조가 갖는 명시적인 트레이드오프로 두었습니다.

- **본 시스템 기준의 선택:** 예상 사용자 규모와 트래픽 패턴에서는 극단적인 고TPS보다 신규 도메인 확장성과 권한 로직의 일관된 관리가 더 중요하다고 판단하여 단일 권한 테이블을 선택했습니다. 추후 실제 병목이 관찰되면 조회 패턴별 인덱스 재설계나 캐시 적용 등을 검토할 수 있습니다.

### 3-2. 권한 상속 구조 설계 (Relationship-Based Access Control)
자원 간의 `부모-자식 관계`를 기반으로 권한이 상속되는 구조를 설계했습니다. 시스템 관점에서는 타겟 자원부터 조상으로 거슬러 올라가는 **상향식 권한 탐색**을 수행하며, 사용자 관점에서는 상위 자원의 권한이 하위로 누적되는 **하향식 권한 적용** 효과를 갖습니다.

**[1] 권한 상속 및 누적 메커니즘**
```text

🗄️ 최상위 드라이브 ── 부여된 권한: [User A : OWNER]
 │
 ├── 📁 상위 폴더 (폴더_1) ── 부여된 권한: [User B : EDITOR]
 │    │ 
 │    ├── 📁 하위 폴더 (폴더_1_1) ── 부여된 권한: 없음
 │    │    │
 │    │    ├── 📄 파일_1.pdf ── 부여된 권한: [User C : VIEWER] (특정 파일 단건 공유)
 │    │    │      ✅ 최종 권한: User A(OWNER), User B(EDITOR), User C(VIEWER)
 │    │    │
 │    │    └── 📄 파일_2.txt ── 부여된 권한: 없음
 │    │           ✅ 최종 권한: User A(OWNER), User B(EDITOR)
 │    │
 │    └── 📄 파일_3.log ── 부여된 권한: 없음
 │           ✅ 최종 권한: User A(OWNER), User B(EDITOR)
 │
 └── 📁 상위 폴더 (폴더_2) ── 부여된 권한: [Group A : VIEWER]
      │ 
      └── 📄 파일_4.xlsx ── 부여된 권한: [User D : EDITOR] (하위 자원에서 권한 격상)
             ✅ 최종 권한: User A(OWNER), Group A(VIEWER), User D(EDITOR)
```

<br>

**[2] 권한 상속 및 검증 흐름**

```mermaid
flowchart TD
    %% 스타일 정의
    classDef user fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;
    classDef resource fill:#e3f2fd,stroke:#1565c0,stroke-width:2px;
    classDef db fill:#fce4ec,stroke:#c2185b,stroke-width:2px;
    classDef resultOn fill:#fff9c4,stroke:#fbc02d,stroke-width:2px;
    classDef resultOff fill:#ffebee,stroke:#c62828,stroke-width:2px;

    Client(["👤 요청 주체<br>(User 본인 + 속한 Group들)"]):::user

    subgraph Step1 ["1. 상향식 경로 ID 추출"]
        direction BT
        File["📄 타겟 자원 ID<br>(예: 파일)"]:::resource
        Folder["📁 조상 폴더 ID 리스트<br>(예: 상위 폴더들)"]:::resource
        Drive["🗄️ 최상위 루트 ID<br>(예: 드라이브)"]:::resource
        
        File -. "속함" .-> Folder -. "속함" .-> Drive
    end

    Client -->|"접근 요청"| File

    subgraph Step2 ["2. 상속 권한 일괄 검증"]
        DB[("permission_tb 조회 (fetchFirst)<br><br>WHERE (target_type, target_id) = 해석된 대상 계층 중 하나<br>AND (subject_type, subject_id) = 요청 주체 중 하나<br>AND role_type IN (허용된 권한들)")]:::db
    end

    File ===>|"ID 전달"| DB
    Folder ===>|"상위 ID 리스트 전달"| DB
    Drive ===>|"ID 전달"| DB

    subgraph Step3 ["3. 최종 인가 결정"]
        Check{"조회 결과<br>(fetchFirst != null)"}
        Check -- "Yes<br>(자신 또는 조상 중 하나라도 권한 일치)" --> Pass["✅ 접근 허용 (인가)"]:::resultOn
        Check -- "No<br>(어떤 계층에서도 권한 찾지 못함)" --> Fail["❌ 권한 부족 (예외 발생)"]:::resultOff
    end

    DB --> Check
```

- **계층 탐색 재귀 쿼리 방지:** `PermissionQueryRepository.hasInheritedAccess`에서 파싱된 조상들의 ID 리스트(`pathIds`)와 최상위 드라이브 코드(`driveCode`)를 `IN` 절로 전달합니다. 이를 통해 트리의 깊이와 무관하게 **한 번의 쿼리로 상속된 권한을 검증**합니다.

- **도메인 분리 및 SRP 준수:** `HierarchyPathResolver` 인터페이스를 도입하여 자원의 타입(`TargetType`)에 따라 폴더(`DriveFolder`)와 파일(`DriveFile`)의 계층 해석 책임을 분리했습니다.

- **조직 단위(Group) 권한 확장을 고려한 설계:** 현재는 유저 개인 단위(`SubjectType.USER`)의 권한 부여를 기본으로 하고 있으나, `QueryDSL` 검증 로직 내부에는 `SubjectType.GROUP`과 `IN (유저의 소속 그룹 리스트)` 조건이 `OR` 절로 통합되어 있습니다. 향후 사내 조직 및 그룹 관리 시스템 연동이 완료되면, **유저의 소속 그룹 ID 리스트 파라미터 주입으로 개인 권한과 그룹 권한을 단일 쿼리로 동시 검증**할 수 있습니다.

- **SpEL 파싱 오버헤드 제거 (로컬 캐시 적용):** AOP 단에서 동적 타겟 ID를 추출할 때 사용하는 Spring Expression Language(SpEL)의 파싱 비용을 줄이기 위해, `ConcurrentHashMap` 기반의 인메모리 로컬 캐시를 적용했습니다.

- **주체 기반 권한 격리:** 타겟 계층(자신 및 조상)의 권한을 조회할 때 `subject_type`과 `subject_id` 조건이 결합되어 있습니다. 이를 통해 타 유저의 권한이 현재 유저에게 잘못 상속되어 조회되는 것을 방지합니다.

### 3-3. AOP 기반 선언적 권한 검증 (관심사의 분리)
권한 검증 로직이 비즈니스 코드에 침투하는 것을 막기 위해, 커스텀 어노테이션 `@RequirePermission`과 `PermissionAspect`를 구현하여 권한 검증을 횡단 관심사로 분리했습니다. PermissionService가 중심이 되어 도메인의 특성(계층 유무)에 따라 검증 방식을 동적으로 라우팅합니다.

**[AOP 기반 선언적 권한 검증 시퀀스 다이어그램]**
```mermaid
sequenceDiagram
    autonumber
    actor Client as Client
    participant API as Controller<br>(@RequirePermission)
    participant AOP as PermissionAspect
    participant Svc as PermissionService
    participant Resolver as HierarchyPathResolver
    participant Repo as PermissionQueryRepository

    Client->>AOP: GET /api/drives/folders/{folderId}<br>(Controller 메서드 호출 가로채기)
    
    %% 1. Intercept
    note over AOP, Svc: 1. Intercept & Permission Check
    AOP->>Svc: @RequirePermission 메타데이터와<br>타겟 ID·세션 유저 정보 전달
    
    %% 2. Delegate & Branching
    note over Svc, Repo: 2. Delegate & Branching (Resolver 지원 여부)
    
    alt A. Resolver 지원 도메인 (예: DriveFolder, DriveFile)
        Svc->>Resolver: 계층 파싱 요청 (resolve)
        Resolver-->>Svc: 자신 및 상위 조상 ID 파싱 반환
        Svc->>Repo: hasInheritedAccess()<br>[상속 권한 1-Query 검증]
        Repo-->>Svc: Query Result (Boolean)
        
    else B. 미지원 단일 도메인 (예: Project, Prompt 등)
        Svc->>Repo: hasAnyRequiredRole()<br>[단건 권한 검증]
        Repo-->>Svc: Query Result (Boolean)
    end

    %% 3. Result Validation
    note over AOP, Svc: 3. Result Validation (True / False)
    Svc-->>AOP: 최종 검증 결과 반환
    
    alt True (접근 허용)
        AOP->>API: proceed()
        API-->>Client: 비즈니스 로직 실행 후 성공 응답
    else False (접근 불가)
        AOP-->>Client: BusinessException (PERMISSION_DENIED)
    end
```

- **다건 처리 전략:** **단건 자원**에 대한 접근 제어는 AOP를 통해 위처럼 처리되며, **다건 자원**(`예: 폴더와 파일들 삭제 및 이동` 등) 검증 시에는 퍼사드 계층에서 비즈니스 로직 진입 전 다건 권한 검증 서비스 함수(`예: drivePermissionService.validateBoundaryAndAccess`)를 호출하도록 했습니다.

### 3-4. 동적 계층 해석기와 전략 패턴 적용
`PermissionService` 내에서 대상 자원이 계층 구조를 가지는지(폴더/파일), 단일 구조인지(프로젝트/프롬프트)를 `if-else` 분기문으로 하드코딩하지 않고, **전략 패턴**을 통해 확장성 있게 설계했습니다.

- **확장 지점 분리(OCP 지향):** `HierarchyPathResolver` 인터페이스를 정의하고, 자원 타입(`TargetType`)별 구현체(`DriveFolderHierarchyResolver`, `DriveFileHierarchyResolver` 등)를 분리했습니다.

- **동적 라우팅 지원:** `PermissionService.checkPermission` 호출 시, 등록된 빈(Bean) 리스트 중 `resolver.supports(targetType)`를 만족하는 구현체가 런타임에 동적으로 매핑됩니다.

- **유연한 폴백:** Resolver가 존재하는 도메인은 계층 기반 상속 검증(`hasInheritedAccess`)을 수행하고, 없는 도메인은 일반 RBAC 검증(`hasAnyRequiredRole`)으로 폴백합니다. 계층형 도메인이 추가되더라도 공통 권한 검증 흐름의 수정을 최소화하고 Resolver 구현체를 추가하는 방식으로 확장할 수 있습니다.

---
<br>

## 4. 공유 링크 시스템
관리자가 사용자를 일일이 검색해 권한을 부여하는 번거로움을 줄이기 위해, 토큰 기반 URL을 통해 정해진 권한을 획득할 수 있도록 한 공유 링크 시스템입니다.

### 4-1. 권한 부여 로직의 자동화 (초대장 메커니즘)
- **동작 방식:** 방장(생성자)이 자원에 대한 공유 링크를 생성하면 고유 토큰이 발급됩니다. 다른 유저가 로그인 상태에서 이 링크(API)를 호출하면, 토큰을 검증한 뒤 `PermissionService.grantPermission()`을 호출하여 해당 유저에게 자원 접근 권한을 부여합니다.

- **도메인별 맞춤 권한 부여:** 자원의 성격에 따라 부여되는 기본 권한을 다르게 설정했습니다. (`determineRoleByTarget`)
  - **드라이브/폴더/파일:** 정보 열람이 주 목적이므로 `VIEWER` (읽기 권한) 부여
  - **채팅방:** 구성원 간 소통과 협업이 목적이므로 `EDITOR` (쓰기 권한) 부여

### 4-2. 기존 다형성 구조 재사용
- 공유 링크 테이블(`ShareLinkEntity`)을 설계할 때, 권한 시스템에서 사용했던 `TargetType`과 `TargetId` 복합 구조를 동일하게 적용했습니다.

- 본래 채팅방 공유만을 위해 기획된 기능이었으나, 추후 다른 도메인으로의 확장성을 고려해 설계했습니다.

- 드라이브나 채팅방뿐만 아니라 향후 프로젝트, 위키 등 신규 자원이 추가되더라도 **공유 링크 테이블 스키마를 변경하지 않고 공통 공유 흐름을 재사용**할 수 있도록 했습니다. 단, 신규 도메인의 기본 권한 정책이나 검증 규칙이 다르면 해당 정책을 연결하는 코드는 추가될 수 있습니다.

### 4-3. 공유 링크의 생명주기 관리
무분별한 권한 획득을 막기 위해 공유 링크의 수명과 상태를 관리합니다.
- **유효 기간 (TTL):** 링크 생성 시 `expiresAt`을 설정하여, 만료일이 지난 링크로 접근할 경우 예외(`SYSTEM_BAD_REQUEST`)를 발생시키고 권한 부여를 차단합니다.

- **생성자 제어권 보장:** `ShareLinkController`를 통해 자신이 생성한 링크 목록을 도메인별(`TargetType`)로 조회할 수 있으며, 필요 없어진 링크는 단건/다건 혹은 일괄 삭제하여 이후 링크를 통한 신규 권한 획득을 차단할 수 있습니다.

- **권한과 링크 수명의 분리:** 현재 구조에서는 링크 사용 시 `PermissionService.grantPermission()`으로 권한 레코드가 생성되므로, 링크 만료·삭제는 링크 자체를 무효화할 뿐 이미 획득한 권한까지 자동 회수하지는 않습니다. 권한 회수가 필요한 정책이라면 링크와 부여 권한의 연관 관계를 별도로 추적해야 합니다.

---
<br>

## 5. 드라이브 시스템 성능 최적화 검증

> **측정값 해석 범위**  
> 아래 수치는 로컬 테스트 환경에서 동일 조건의 기존/개선 구현을 비교한 결과입니다. `EXPLAIN ANALYZE`, `Spring StopWatch`, `Hibernate Statistics`로 실행 시간과 쿼리 수를 확인했으며, 절대적인 운영 환경 성능이나 p95/p99 지연을 의미하지는 않습니다. 따라서 성능 수치는 **동일 테스트 환경 내 상대 비교**로 해석합니다.

### 5-1. 대용량 계층 구조 탐색 성능 향상 검증 (31.4ms → 2.7ms)

#### 5-1-1. 설계 배경 및 가설 
- 드라이브 시스템의 핵심 도메인 특성은 **쓰기(Write)보다 읽기(Read)의 빈도가 압도적으로 높다**는 것입니다. 

- 초기 DB 스키마 설계 시, 데이터 무결성과 직속 하위 폴더 탐색을 위해 인접 목록 모델(Adjacency List, `parent_id` 참조)을 구성했습니다.

- 하지만 이 방식은 특정 폴더 **하위 및 상위의 모든 계층 자원(N-Depth)**을 탐색할 때 재귀 쿼리(`Recursive CTE`)가 유발되며, 트리의 깊이(`Depth`)가 깊어질수록 반복적인 조인과 임시 테이블(`WorkTable`) 생성으로 인한 I/O 병목이 발생할 것으로 예상했습니다.

- 이에 따라 쓰기 비용(경로 업데이트)을 감수하더라도, 조회 성능을 끌어올릴 수 있는 **경로 열거(Path Enumeration) 패턴**을 도입하기로 결정했습니다.

#### 5-1-2. 설계 구현 및 인덱스 
- **Path-based 스키마 설계:** 모든 자원에 누적 논리 경로를 저장하는 `logical_path` (예: `/root-id/sub-id/`) 컬럼을 추가하여, 단일 `LIKE '/target-id/%'` 연산만으로 하위 자원을 모두 식별할 수 있도록 구성했습니다.

- **Pattern Matching 인덱스 (`varchar_pattern_ops`):** PostgreSQL의 일반 B-Tree 인덱스는 collation 설정에 따라 prefix `LIKE` 조건에서 기대한 인덱스 탐색이 어려울 수 있습니다. `logical_path`에 `varchar_pattern_ops`를 적용하여 locale 기반 정렬 규칙 대신 문자열 패턴 비교에 적합한 연산자 클래스를 사용하도록 했고, 실제 실행계획에서 `Bitmap Index Scan`이 선택되는 것을 확인했습니다.

- **부분 인덱스 적용:** Soft Delete(`is_deleted = true`)된 데이터는 일반적인 폴더 트리 탐색 대상에서 제외됩니다. 따라서 `WHERE is_deleted = false` 조건을 단 부분 인덱스를 생성하여, 인덱스 트리의 크기를 줄였습니다.

#### 5-1-3. 성능 검증
**[테스트 환경 및 데이터 셋업]**
* **데이터 규모:** 더미 데이터 총 `58만` 건 구축 (`폴더 10만 건, 파일 48만 건`)
* **테스트 대상:** 폴더 깊이가 `2`인 폴더 중 하위 자원 총 `23,436`개를 보유한 폴더. 아래 `drive_folder_tb` 실행계획 비교는 이 가운데 **하위 폴더 3,906개**를 조회하는 쿼리를 대상으로 수행했습니다.
* **측정 도구:** PostgreSQL `EXPLAIN ANALYZE`

<br>

**[실행 계획 분석 비교]**
* **대조군 (Recursive CTE):**
```sql
EXPLAIN ANALYZE
WITH RECURSIVE folder_tree AS (
    SELECT id, parent_folder_id, name, logical_path
    FROM drive_folder_tb 
    WHERE id = '59ff3411-809a-4048-9f7e-2a017dde5b0e'
      AND is_deleted = false
    
    UNION ALL
    
    SELECT f.id, f.parent_folder_id, f.name, f.logical_path
    FROM drive_folder_tb f
    INNER JOIN folder_tree t ON f.parent_folder_id = t.id
    WHERE f.is_deleted = false
)
SELECT * FROM folder_tree;
```


```sql
CTE Scan on folder_tree  (cost=2430.20..2440.22 rows=501 width=2064) (actual time=0.044..31.053 rows=3906 loops=1)
  CTE folder_tree
    ->  Recursive Union  (cost=0.42..2430.20 rows=501 width=385) (actual time=0.042..29.654 rows=3906 loops=1)
          ->  Index Scan using drive_folder_tb_pkey on drive_folder_tb  (cost=0.42..8.44 rows=1 width=385) (actual time=0.041..0.042 rows=1 loops=1)
                Index Cond: ((id)::text = '59ff3411-809a-4048-9f7e-2a017dde5b0e'::text)
                Filter: (NOT is_deleted)
          ->  Nested Loop  (cost=4.46..241.68 rows=50 width=385) (actual time=3.345..4.794 rows=651 loops=6)
                ->  WorkTable Scan on folder_tree t  (cost=0.00..0.20 rows=10 width=516) (actual time=0.000..0.124 rows=651 loops=6)
                ->  Bitmap Heap Scan on drive_folder_tb f  (cost=4.46..24.10 rows=5 width=385) (actual time=0.005..0.005 rows=1 loops=3906)
                      Recheck Cond: (((t.id)::text = (parent_folder_id)::text) AND (NOT is_deleted))
                      Heap Blocks: exact=1027
                      ->  Bitmap Index Scan on uk_active_subfolder_name  (cost=0.00..4.46 rows=5 width=0) (actual time=0.005..0.005 rows=1 loops=3906)
                            Index Cond: ((parent_folder_id)::text = (t.id)::text)
Planning Time: 0.259 ms
Execution Time: 31.440 ms
```

Recursive CTE의 `WorkTable`을 사용하면서 하위 폴더를 단계적으로 확장했고, 자식 탐색을 위한 내부 스캔이 `loops=3906`까지 반복되었습니다. (소요 시간: **31.440 ms**)

<br>

* **실제 적용군 (Path Enumeration):**
```sql
EXPLAIN ANALYZE
SELECT id, parent_folder_id, name, logical_path
FROM drive_folder_tb 
WHERE logical_path LIKE '/bef81acd-b8c5-2352-8280-37cdf5536098/7abc839b-4b1c-45eb-9e98-fc9b9075b805/59ff3411-809a-4048-9f7e-2a017dde5b0e/%'
  AND is_deleted = false;
```

```sql
Bitmap Heap Scan on drive_folder_tb  (cost=1048.21..6872.69 rows=3946 width=385) (actual time=0.875..2.650 rows=3906 loops=1)
  Recheck Cond: (NOT is_deleted)
  Filter: ((logical_path)::text ~~ '/bef81acd-b8c5-2352-8280-37cdf5536098/7abc839b-4b1c-45eb-9e98-fc9b9075b805/59ff3411-809a-4048-9f7e-2a017dde5b0e/%'::text)
  Heap Blocks: exact=261
  ->  Bitmap Index Scan on idx_active_logical_path  (cost=0.00..1047.22 rows=3868 width=0) (actual time=0.843..0.843 rows=3906 loops=1)
        Index Cond: (((logical_path)::text ~>=~ '/bef81acd-b8c5-2352-8280-37cdf5536098/7abc839b-4b1c-45eb-9e98-fc9b9075b805/59ff3411-809a-4048-9f7e-2a017dde5b0e/'::text) AND ((logical_path)::text ~<~ '/bef81acd-b8c5-2352-8280-37cdf5536098/7abc839b-4b1c-45eb-9e98-fc9b9075b805/59ff3411-809a-4048-9f7e-2a017dde5b0e0'::text))
Planning Time: 0.182 ms
Execution Time: 2.755 ms
```
  * 루프가 발생하지 않고, **Bitmap Index Scan**을 통해 조건에 맞는 인덱스 블록만 추출함을 확인했습니다. (소요 시간: **2.755 ms**)

<br>

#### 5-1-4. 검증 결과

```mermaid
xychart-beta
    title "계층 구조 탐색 쿼리 실행 시간 (단위: ms)"
    x-axis ["기존 (Recursive CTE)", "개선 (Path Enumeration)"]
    y-axis "실행 시간 (ms)" 0 --> 35
    bar [31.4, 2.7]
```

| 설계 지표 | 일반적 인접 목록 (대조군) | 경로 열거 패턴 (실제 적용군) | 도입 성과 |
| :--- | :--- | :--- | :--- |
| **실행 시간 (DB 쿼리)** | 31.440 ms | 2.755 ms | 약 11배 성능 우위 |
| **DB 스캔 방식** | Recursive Union + 반복 Nested Loop | Bitmap Index Scan + Bitmap Heap Scan | 반복 조인 및 Heap 접근 감소 |


---

<br>

### 5-2. 폴더 임베딩 상태 동기화 성능 최적화: 쿼리 39회 → 4~5회

#### 5-2-1. 문제 상황: 상향식 전파의 구조적 한계
파일 임베딩 완료 시, 해당 파일이 속한 폴더부터 최상위 루트까지 모든 조상 폴더들의 **임베딩 완료 여부**를 실시간으로 갱신해야 합니다.

* **기존 방식 (재귀적 동기화):**
```java
List<String> folderIds = new ArrayList<>();
String[] pathParts = logicalPath.split("/");

// 파싱 및 역순 정렬 (자식 -> 부모 -> 조상)
for (int i = pathParts.length - 1; i >= 0; i--) {
    if (!pathParts[i].isEmpty()) {
        folderIds.add(pathParts[i]);
    }
}

// 상태 업데이트
for (String folderId : folderIds) {
    boolean hasIncompleteFile = driveFileQueryRepository.existsNotCompletedFileInFolder(folderId);

    boolean hasIncompleteSubFolder = driveFolderQueryRepository.existsNotCompletedSubFolder(folderId);

    boolean isComplete = !hasIncompleteFile && !hasIncompleteSubFolder;

    driveFolderQueryRepository.updateAllEmbeddedStatus(folderId, isComplete);
}
```
  * 자식 폴더에서 부모 폴더 ID를 참조하여 한 단계씩 거슬러 올라가는 방식.
  * 조상 폴더 하나를 검증할 때마다 `[미완료 파일 체크(SELECT) → 미완료 자식 폴더 체크(SELECT) → 상태 갱신(UPDATE)]`의 3단계 과정이 반복됩니다.
  * 결과: 루트의 Depth를 0으로 두면 Depth $D$의 폴더에서 루트까지 상태를 확인해야 하는 폴더 수는 $D+1$개입니다. 따라서 기존 방식은 약 $3(D+1)$회의 쿼리를 발생시킵니다. 테스트 환경의 `Depth 12`에서는 **13개 폴더 × 3회 = 39회**의 DB 쿼리가 발생했습니다.

#### 5-2-2. 해결 방안: 메모리 연산 & Bulk Update
매 단계 DB를 조회하는 대신, 임베딩 상태값 변화 영향권에 있는 모든 조상 폴더를 메모리에 로드하여 연산한 뒤 결과를 일괄 반영하도록 구조를 변경했습니다.

```java
List<DriveFolderEntity> folders = driveFolderQueryRepository.findByIds(new ArrayList<>(folderIds));
Set<String> allTargetFolderIds = folders.stream()
        .flatMap(f -> f.getAncestorFolderIds().stream())
        .collect(Collectors.toSet());

if (allTargetFolderIds.isEmpty()) return;

// 미완료 파일을 가진 폴더 아이디들 (변하지 않는 상태)
Set<String> folderIdsWithIncompleteFiles = driveFileQueryRepository.findFolderIdsWithIncompleteFiles(allTargetFolderIds);

// 각 폴더별 임베딩 미완료 자식 폴더의 개수 (자식 폴더 상태가 바뀌면 바뀔 수 있음)
Map<String, Long> incompleteChildCounts = driveFolderQueryRepository.findIncompleteSubFolderCounts(allTargetFolderIds);

// 논리 경로에 있던 폴더 조회 및 정렬 (자식 -> 부모 순)
List<DriveFolderEntity> sortedFolders = driveFolderQueryRepository.findByIds(new ArrayList<>(allTargetFolderIds));
sortedFolders.sort((f1, f2) -> Integer.compare(f2.getDepth(), f1.getDepth()));

// 폴더 임베딩 상태 bulk 연산 시킬 폴더 ID 목록
List<String> folderIdsToMarkComplete = new ArrayList<>();
List<String> folderIdsToMarkIncomplete = new ArrayList<>();

// 자식 -> 부모 -> 조상 순서대로 폴더 임베딩 완료 상태 업데이트 및 업데이트할 폴더 아이디 수집
evaluateStatusAndCollectUpdates(sortedFolders, folderIdsWithIncompleteFiles, incompleteChildCounts, folderIdsToMarkComplete, folderIdsToMarkIncomplete);

// bulk로 일괄 업데이트
if (!folderIdsToMarkComplete.isEmpty()) {
    driveFolderRepository.bulkUpdateAllEmbeddedStatus(folderIdsToMarkComplete, true);
}
if (!folderIdsToMarkIncomplete.isEmpty()) {
    driveFolderRepository.bulkUpdateAllEmbeddedStatus(folderIdsToMarkIncomplete, false);
}
```

   * **조상 폴더 집합 일괄 조회:** `logical_path`를 파싱하여 루트까지의 조상 ID 집합을 만들고, 이후 필요한 폴더 엔티티·미완료 파일 보유 폴더·미완료 자식 폴더 수를 각각 집합 단위 쿼리로 조회합니다. 폴더마다 반복 조회하는 N+1 형태를 제거하는 것이 핵심입니다.
   * **메모리 기반 상향식 전파:** 로드된 폴더들을 Depth 역순(`자식 → 부모`)으로 정렬하여 메모리 내에서 상태를 평가합니다. 자식 폴더의 상태가 결정되면 부모 폴더의 미완료 카운트를 갱신하여 DB 조회 없이 연쇄적인 상태 판별을 수행합니다.
   * **최종 상태 Bulk Update:** 연산 결과, 실제로 상태 변경(`True` 또는 `False`)이 필요한 폴더 ID만 필터링하여 최대 2회의 Bulk Update 쿼리로 모든 조상의 상태를 한 번에 갱신합니다.


#### 5-2-3. 검증 결과

```mermaid
xychart-beta
    title "상태 동기화 시 DB 쿼리 발생 횟수"
    x-axis ["기존 (재귀적 조회)", "개선 (In-Memory 연산 후 일괄 갱신)"]
    y-axis "쿼리 횟수" 0 --> 45
    bar [39, 5]
```

* **테스트 환경:** 총 폴더 `8,191`개, 파일 `16,380`개, 최대 깊이 12의 트리 구조
* **테스트 시나리오:** 최하단(`Depth 12`) 파일의 상태 변경을 루트(`Depth 0`)까지 전파. 부모-자식 간선은 12단계이며 상태 판단 대상 폴더는 총 13개입니다.
* **측정 도구:** JUnit 5, Spring StopWatch, Hibernate Statistics

| 성능 지표 | 기존 방식 <br> (Recursive DB I/O) | 개선 방식 <br> (In-Memory 연산 후 Bulk Update) | 개선 수치 |
| :--- | :--- | :--- | :--- |
| **쿼리 발생 횟수** | 39회 | **4~5회 (해당 시나리오 실측)** | **최대 87.1% 절감** |
| **처리 시간 (로컬 환경)** | 0.212 sec | **0.101 sec** | **52.3% 단축** |

<br>

**[최적화 성과 요약]**
- **DB 쿼리 발생 횟수 감소:** 폴더마다 반복하던 조회/갱신을 집합 단위 조회와 Bulk Update로 전환하여, `Depth 12` 테스트 시나리오에서 쿼리 수를 `39회`에서 `4~5회`로 줄였습니다. 제시한 구조에서는 완료/미완료 상태가 동시에 변경되는 경우 Bulk Update가 각각 실행될 수 있으므로, 일반적인 실행 경로의 쿼리 상한은 테스트 데이터의 상태 분포에 따라 달라질 수 있습니다.

- **동기화 처리 시간 단축:** 조상 폴더의 상태 판별을 DB I/O 대신 애플리케이션 메모리 내 연산으로 전환하고 결과를 Bulk Update로 일괄 반영하여, 단일 동기화 처리 시간을 물리적 네트워크 지연이 없는 로컬 환경 기준으로 0.212초에서 0.101초로 단축했습니다.

- **트랜잭션 점유 시간 완화:** 해당 로직은 `1초` 주기의 `벡터 임베딩 스케줄러` 및 드라이브 내 쓰기 작업(`업로드`, `이동`, `삭제` 등)에서 트리거되는 동기화 로직입니다. 해당 동기화 로직의 최적화를 통해 시스템 전반의 DB 트랜잭션 점유 및 커넥션 부하를 줄였습니다.

---

### 5-3. 대규모 계층 구조 이동 성능 최적화: 쿼리 발생 횟수 상수 횟수로 최적화

#### 5-3-1. 문제 상황
드라이브 시스템에서 경로 열거 패턴을 사용하면 조회 성능은 상승하지만, **특정 폴더를 다른 위치로 이동할 때 하위 모든 자손들의 `logical_path`와 `depth`를 전부 수정해야 하는 쓰기 비용**이 발생합니다.

```java
// 하위 폴더를 모두 조회하여 메모리(영속성 컨텍스트)에 올린 뒤 하나씩 경로 수정
List<DriveFolderEntity> descendants = driveFolderQueryRepository.findDescendantFoldersByPath(originalPath);
for (DriveFolderEntity child : descendants) {
    String newPath = child.getLogicalPath().replaceFirst(originalPath, newPath);
    child.updatePathInfo(newPath, newDepth); // 더티 체킹으로 인한 개별 UPDATE 발생
}
```

* **$O(N)$ 쿼리 발생:** 하위 자원의 개수에 정비례하여  `UPDATE` 쿼리가 발생합니다. 하위 폴더가 약 2만 개일 경우, 19,500여 번에 달하는 쿼리가 발생합니다.
* **DB 커넥션 점유:** 단일 요청이 DB 트랜잭션을 오랫동안 점유하면서, 동시 접속자가 늘어나면 커넥션 풀 고갈 위험이 있습니다.
* **메모리 및 연산 오버헤드:** 이동할 폴더의 하위 폴더들을 모두 영속성 컨텍스트(메모리)에 올려 객체의 상태를 하나씩 변경합니다.

#### 5-3-2. 해결 방안
JPA의 변경 감지 대신, **DB 내장 함수(`REPLACE`)와 QueryDSL을 활용한 Bulk Update**로 수정했습니다.

```sql
-- 애플리케이션 메모리 로딩을 배제하고, DB 레벨에서 경로 문자열 치환 연산 일괄 수행
UPDATE drive_folder_tb 
SET logical_path = REPLACE(logical_path, '/old-path/', '/new-path/'),
    depth = depth + :depthDifference
WHERE logical_path LIKE '/old-path/%';
```

* **DB 내장 함수 활용:** 애플리케이션 단에서 문자열을 조작하지 않고, SQL의 `REPLACE` 함수를 활용하는 Bulk 쿼리를 작성하여 하위 폴더들의 `logical_path`와 `depth`를 한 번의 쿼리로 일괄 갱신했습니다.


#### 5-3-3. 검증 결과
데이터 규모를 4단계로 나누어 테스트하였습니다.
* **측정 도구:** JUnit 5 + Spring StopWatch, Hibernate Statistics

| 이동 대상 하위 폴더 규모 | 기존 방식 (JPA 재귀) 처리 시간 | 개선 방식 (Bulk) 처리 시간 | 기존 방식 발생 쿼리 | 개선 방식 발생 쿼리 |
| :--- | :--- | :--- | :--- | :--- |
| **157개** | 0.41 초 | **0.12 초** | 159 회 | **7 회** |
| **782개** | 0.93 초 | **0.22 초** | 784 회 | **7 회** |
| **3,907개** | 2.16 초 | **0.30 초** | 3,909 회 | **7 회** |
| **19,532개**| 5.63 초 | **1.20 초** | 19,534 회 | **7 회** |

* *`참고`: 개선 방식의 7회 쿼리는 권한 및 대상 확인을 위한 검증용 SELECT 5회와 이동 대상 최상위 폴더 자체를 갱신하는 단건 UPDATE 1회, 그리고 하위 자손 폴더 전체의 경로와 깊이를 치환하는 Bulk UPDATE 1회로 구성됩니다.*

<br>

**[최적화 성과 요약]**
- **DB 통신 비용(Network RTT) 최적화:** 데이터가 2만 건으로 증가해도 발생하는 DB 쿼리를 상수 횟수인 7회로 고정시켜, 애플리케이션과 DB 간의 통신 횟수를 99.96% 절감했습니다.
- **처리 시간 단축 및 장기 점유 가능성 완화:** `Spring StopWatch`로 측정한 요청 처리 시간을 5.63초에서 1.20초로 78.7% 단축했습니다. 이는 트랜잭션과 DB 커넥션이 장시간 점유될 가능성을 낮추지만, 별도의 커넥션 풀 메트릭으로 실제 점유 시간을 직접 측정한 값은 아닙니다.
- **영속성 컨텍스트 메모리 부하 감소:** 약 2만 개의 대상 엔티티를 애플리케이션 메모리에 로드해 변경하던 과정을 제거하여 heap 사용량과 GC 부담, 대용량 처리 시 OOM 위험을 낮췄습니다.
- **복잡도 해석:** 애플리케이션↔DB 간 쿼리 왕복 횟수는 7회로 일정해졌지만, DB 내부에서는 대상 행 수만큼 실제 UPDATE 작업과 인덱스 유지 비용이 발생하므로 전체 갱신 비용 자체가 $O(1)$이 되는 것은 아닙니다.

---

<br>

### 5-4. 대규모 계층 구조의 Soft Delete 성능 최적화: 쿼리 발생 횟수 상수 횟수로 최적화

#### 5-4-1. 문제 상황
드라이브 서비스에서 상위 폴더를 삭제하면 해당 폴더를 포함하여 하위의 모든 자식 폴더와 파일들의 상태를 `is_deleted = true`로 변경하는 **Soft Delete** 작업이 필요합니다. 기존 로직은 JPA 변경 감지에 의존하여 성능 병목을 유발했습니다.

```java
// 하위 자원을 찾기 위해 엔티티 그래프를 탐색하며 N번의 재귀 호출 발생
private void deleteFolderRecursively(DriveFolderEntity folder) {
    folder.delete();
    for (DriveFileEntity file : folder.getFiles()) file.delete(); 
    for (DriveFolderEntity sub : folder.getChildFolders()) deleteFolderRecursively(sub);
}
```

* **$O(N)$ 쿼리 발생:** 하위 자원에 비례하여 선형적으로 쿼리가 증가합니다. 하위 폴더가 약 2만 개일 경우, 약 2만 번에 달하는 쿼리가 발생합니다.
* **DB 커넥션 점유:** 단일 요청이 DB 트랜잭션을 오랫동안 점유하면서, 동시 접속자가 늘어나면 커넥션 풀 고갈 위험이 있습니다.
* **메모리 및 연산 오버헤드:** 삭제할 폴더의 하위 폴더들을 모두 영속성 컨텍스트(메모리)에 올려 객체의 상태를 하나씩 변경합니다.

#### 5-4-2. 해결 방안
JPA의 변경 감지 대신, **QueryDSL을 활용한 Bulk Update**로 수정했습니다.

```sql
-- 개념적 SQL: 애플리케이션 메모리에 엔티티를 모두 적재하지 않고 DB에서 집합 단위 갱신
UPDATE drive_folder_tb
SET is_deleted = true
WHERE logical_path LIKE '/root-id/%';

UPDATE drive_file_tb
SET is_deleted = true
WHERE folder_id IN (
    SELECT id
    FROM drive_folder_tb
    WHERE logical_path LIKE '/root-id/%'
);
```

* **Path-based 필터링:** 삭제 대상 폴더의 경로가 `/A/B/`라면 하위 폴더의 `logical_path`는 `/A/B/%` 형태를 가지므로, 경로 prefix를 기준으로 삭제 범위를 집합 단위로 식별할 수 있습니다.
* **폴더/파일 각각 Bulk Update:** JPA 엔티티를 하나씩 영속성 컨텍스트에 올리는 대신, 폴더와 파일을 **각각 1회의 Bulk Update**로 처리하도록 변경했습니다. 위 SQL은 동작 원리를 나타낸 개념적 형태이며, 실제 구현에서는 QueryDSL Bulk Update로 동일한 삭제 범위를 표현합니다.


#### 5-4-3. 검증 결과
데이터 규모를 4단계로 나누어 테스트하였습니다.
* **측정 도구:** JUnit 5, Spring StopWatch, Hibernate Statistics

| 삭제 대상 하위 폴더 규모 | 기존 방식 (JPA 재귀) 처리 시간 | 개선 방식 (Bulk) 처리 시간 | 기존 방식 발생 쿼리 | 개선 방식 발생 쿼리 |
| :--- | :--- | :--- | :--- | :--- |
| **156개** | 0.41 초 | **0.11 초** | 167 회 | **5 회** |
| **781개** | 0.86 초 | **0.17 초** | 806 회 | **5 회** |
| **3,906개** | 2.24 초 | **0.19 초** | 3,995 회 | **5 회** |
| **19,531개**| 7.88 초 | **0.70 초** | 19,934 회 | **5 회** |

* *`참고`: 개선 방식의 5회 쿼리는 권한 및 대상 확인을 위한 검증용 SELECT 3회와 삭제할 폴더 하위에 속한 모든 파일과 폴더를 각각 일괄 Soft Delete 처리하는 Bulk UPDATE 2회로 구성됩니다.*

<br>

**[최적화 성과 요약]**
- **DB 통신 비용(Network RTT) 최적화:** 데이터가 2만 건으로 증가해도 발생하는 DB 쿼리를 상수 횟수인 5회로 고정시켜, 애플리케이션과 DB 간의 통신 횟수를 99.97% 감소시켰습니다.
- **처리 시간 단축 및 장기 점유 가능성 완화:** `Spring StopWatch` 기준 처리 시간을 7.88초에서 0.70초로 91.1% 단축했습니다. 이를 통해 단일 삭제 요청이 트랜잭션과 DB 커넥션을 장시간 점유할 가능성을 낮췄습니다.
- **영속성 컨텍스트 메모리 부하 감소:** 약 2만 개의 대상 엔티티를 애플리케이션 메모리에 적재하던 과정을 제거하여 heap 사용량과 GC 부담, 대규모 처리 시 OOM 위험을 낮췄습니다.
- **복잡도 해석:** 애플리케이션↔DB 간 쿼리 왕복 횟수는 5회로 일정하지만, 실제 DB 내부의 행 갱신 비용은 삭제 대상 건수에 비례합니다.

---

<br>

### 5-5. 대규모 임베딩 스케줄러 폴링 쿼리 병목 해소

#### 5-5-1. 문제 상황
드라이브 파일의 벡터 임베딩 처리를 위해 4개의 워커 스레드가 1초 주기로 DB를 조회(`FOR UPDATE SKIP LOCKED`)하여 임베딩 대상 파일을 가져오는 스케줄러 로직을 구현했습니다. 하지만 부하 환경을 가정한 테스트에서 큐가 비어있을 때 조회 병목이 확인되었습니다.

```java
// 기존 조회 로직: 모든 상태에 대해 동일한 정렬 적용 및 isNull 조건 포함
return query.selectFrom(qDriveFileEntity)
        .where(
            qDriveFileEntity.embeddingStatus.eq(status),
            version != null ? qDriveFileEntity.version.isNull().or(qDriveFileEntity.version.lt(version)) : null
        )
        .orderBy(qDriveFileEntity.createdDate.asc())
        .limit(1)
        .setLockMode(LockModeType.PESSIMISTIC_WRITE)
        .fetchOne();
```

* **비선택적 실행계획 확인:** 기존 조건은 `version < :version OR version IS NULL`과 `ORDER BY created_date ASC`가 함께 적용되었습니다. 이 조합이 항상 특정 실행계획을 강제하는 것은 아니지만, **본 테스트 데이터에서는** 옵티마이저가 `(embedding_status, created_date)` 계열 인덱스를 선택했고 `version` 조건은 `Filter`로 남았습니다.
* **대규모 후보 행 스캔:** 처리할 대상이 없는 `COMPLETED / version < 0.5` 조건에서 실행계획의 `Rows Removed by Filter`가 `3,308,760`으로 나타났습니다. 이는 330만 행을 애플리케이션 메모리에 적재했다는 의미가 아니라, 인덱스를 따라 탐색한 후보 행들이 필터 조건에서 제거되었다는 의미입니다.
* **주기적 폴링 부하:** 단일 조회가 약 1.29초가 걸리는 상태에서 4개 워커가 1초 주기로 같은 유형의 탐색을 반복하면 DB CPU/I/O와 커넥션 사용량에 지속적인 부하를 줄 수 있다고 판단했습니다.

#### 5-5-2. 해결 방안
인덱스 분리, 불필요한 방어 로직 제거, 동적 정렬 처리를 통해 쿼리를 최적화했습니다.

```sql
-- 1. 대규모 임베딩 완료 데이터 버전업 검사 방어용 인덱스 추가 분리
CREATE INDEX idx_active_embedding_versions ON drive_file_tb (embedding_status, version) WHERE is_deleted = false;
```
```java
// 2. QueryDSL 로직 개선: isNull 제거 및 동적 ORDER BY 적용
JPAQuery<DriveFileEntity> jpaQuery = query.selectFrom(qDriveFileEntity)
        .where(
            qDriveFileEntity.embeddingStatus.eq(status),
            version != null ? qDriveFileEntity.version.lt(version) : null // IS NULL 제거
        );

// 신규 업로드 처리 시에만 순차 탐색 적용, 대규모 백그라운드 재처리는 정렬 배제
if (status == DriveFileEmbeddingStatus.READY || status == DriveFileEmbeddingStatus.PRIORITIZED) {
    jpaQuery.orderBy(qDriveFileEntity.createdDate.asc());
}
```

* **인덱스 분리 타겟팅:** 상태별 데이터 분포도를 고려하여 순차 처리용 인덱스(`status, created_date`)와 버전/시간 필터링용 인덱스(`status, version`)를 분리했습니다.
* **`IS NULL` 조건 제거:** `version`에 NULL이 존재하지 않는다는 DB 제약 또는 기존 데이터 마이그레이션이 보장된다는 전제에서 불필요한 `isNull()` 분기를 제거했습니다. 단순한 `DEFAULT 1.0`만으로 기존/명시적 NULL까지 방지되는 것은 아니므로, 실제 스키마에서는 `NOT NULL` 여부와 기존 데이터 정리가 함께 확인되어야 합니다.
* **동적 정렬:** 사용자 경험상 처리 순서가 중요한 신규 업로드(`READY`, `PRIORITIZED`)에만 `created_date` 정렬을 적용하고, 대규모 재처리(`COMPLETED`, `FAILED`)에서는 정렬을 제거하여 `(embedding_status, version)` 인덱스를 활용할 수 있는 실행계획을 유도했습니다.

#### 5-5-3. 테스트 환경 및 검증 결과
* **데이터 규모:** 3,308,760 건 (`COMPLETED`, `version = 1.0` 상태)
* **테스트 상황:** 처리할 일감이 0건인 최악의 탐색 조건 (`version < 0.5`)
* **측정 도구:** PostgreSQL `EXPLAIN ANALYZE`

**[실행 계획 분석 비교]**
* **대조군 (튜닝 전: 부적절한 쿼리로 인한 대규모 필터링 발생):**
  * **분석:** `embedding_status = COMPLETED` 조건으로 탐색한 후보 중 `3,308,760`행이 `version` 등의 Filter 조건에서 제거되었고, 대상 0건을 확인하는 데 약 1.29초가 소요되었습니다.

튜닝 전 EXPLAIN ANALYZE 원본

```text
Limit  (cost=0.43..6.21 rows=1 width=2731) (actual time=1293.545..1293.546 rows=0 loops=1)
  ->  LockRows  (cost=0.43..6.21 rows=1 width=2731) (actual time=1293.544..1293.545 rows=0 loops=1)
        ->  Index Scan using idx_active_embedding on drive_file_tb  (cost=0.43..6.20 rows=1 width=2731) (actual time=1293.543..1293.543 rows=0 loops=1)
              Index Cond: ((embedding_status)::text = 'COMPLETED'::text)
              Filter: ((NOT is_deleted) AND ((source_type)::text <> ALL ('{CHAT_ROOM,MANUAL_FILE}'::text[])) AND ((version IS NULL) OR (version < '0.5'::double precision)))
              Rows Removed by Filter: 3308760
Planning Time: 19.655 ms
Execution Time: 1293.579 ms
```

<br>

* **실험군 (튜닝 후: 인덱스 분리, IS NULL 제거, 정렬 조건 제거):**
  * **분석:** 옵티마이저가 `(embedding_status, version)` 인덱스 조건을 직접 사용해 대상이 없음을 빠르게 확인했고, 실행계획상 `Rows Removed by Filter` 없이 0.032ms에 종료되었습니다.

튜닝 후 EXPLAIN ANALYZE 원본

```text
Limit  (cost=0.43..6.21 rows=1 width=2731) (actual time=0.015..0.015 rows=0 loops=1)
  ->  LockRows  (cost=0.43..6.21 rows=1 width=2731) (actual time=0.014..0.014 rows=0 loops=1)
        ->  Index Scan using idx_active_embedding_versions on drive_file_tb  (cost=0.43..6.20 rows=1 width=2731) (actual time=0.013..0.013 rows=0 loops=1)
              Index Cond: ((embedding_status)::text = 'COMPLETED'::text AND (version < '0.5'::double precision))
              Filter: ((NOT is_deleted) AND ((source_type)::text <> ALL ('{CHAT_ROOM,MANUAL_FILE}'::text[])))
Planning Time: 0.174 ms
Execution Time: 0.032 ms
```

| 측정 지표 | 기존 방식 (고정 정렬 + `OR IS NULL`) | 개선 방식 (동적 정렬 + 복합 인덱스 조건 활용) |
| :--- | :--- | :--- |
| **선택된 인덱스** | `idx_active_embedding` (version 조건이 Filter로 남음) | **`idx_active_embedding_versions`** (version이 Index Cond에 포함) |
| **필터링된 행 수 (Rows Removed)** | 3,308,760 개 | **0 개** |
| **총 실행 시간 (Execution Time)** | 1293.579 ms | **0.032 ms** |

<br>

**[최적화 성과 요약]**
- **대규모 후보 행 스캔 제거:** 기존 실행계획에서 `Rows Removed by Filter = 3,308,760`으로 나타나던 후보 행 탐색을 없애고 `version` 조건을 `Index Cond`에 포함시켰습니다.
- **폴링 탐색 시간 단축:** 처리 대상이 없는 최악 조건에서 실행 시간을 `1293.579ms`에서 `0.032ms`로 단축했습니다. 이는 해당 데이터 분포와 로컬 테스트 환경에서의 결과이며, 운영 환경의 절대 지연시간을 의미하지는 않습니다.

---

## 6. 기술 스택

| Category | Skills |
| :--- | :--- |
| **Backend** | <img src="https://img.shields.io/badge/Java_21-007396?style=for-the-badge&logo=java&logoColor=white"> <img src="https://img.shields.io/badge/Spring_Boot_3.5.13-6DB33F?style=for-the-badge&logo=springboot&logoColor=white"> |
| **DB & ORM** | <img src="https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white"> <img src="https://img.shields.io/badge/Spring_Data_JPA-6DB33F?style=for-the-badge&logo=spring&logoColor=white"> <img src="https://img.shields.io/badge/QueryDSL_5.0-0769AD?style=for-the-badge&logo=java&logoColor=white"> |
| **Security** | <img src="https://img.shields.io/badge/Spring_Security-6DB33F?style=for-the-badge&logo=springsecurity&logoColor=white"> <img src="https://img.shields.io/badge/JWT-000000?style=for-the-badge&logo=jsonwebtokens&logoColor=white"> |
