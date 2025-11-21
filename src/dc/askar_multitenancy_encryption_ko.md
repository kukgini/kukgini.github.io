# Askar Multitenancy Mode 데이터 암호화 과정

## 개요

Askar는 multitenancy mode에서 여러 개의 독립적인 profile을 지원합니다. 각 profile은 자신만의 암호화 키를 가지며, 모든 profile의 키는 공통의 Store Key로 암호화되어 저장됩니다. 이 문서는 이러한 구조에서 데이터가 어떻게 암호화되어 저장되는지 설명합니다.

## 키 계층 구조

Askar의 암호화는 3단계 키 계층 구조를 사용합니다:

1. **Store Key (최상위 키)**
   - 모든 profile key를 암호화하는 마스터 키
   - Passphrase + Argon2i KDF 또는 Raw Key로 생성
   - 데이터베이스에 저장되지 않음 (메모리에만 존재)

2. **Profile Key (프로필별 키)**
   - 각 profile마다 고유하게 생성되는 키
   - Store Key로 암호화되어 `profiles` 테이블에 저장
   - 6개의 하위 키를 포함:
     - `ick` (item category key): Category 암호화용
     - `ink` (item name key): Name 암호화용
     - `ihk` (item HMAC key): Item 검색 가능한 암호화용 HMAC
     - `tnk` (tag name key): Tag 이름 암호화용
     - `tvk` (tag value key): Tag 값 암호화용
     - `thk` (tags HMAC key): Tag 검색 가능한 암호화용 HMAC

3. **Derived Keys (파생 키)**
   - Item Value 암호화를 위해 category와 name으로부터 동적으로 생성
   - HMAC-SHA-256을 사용하여 파생

## 데이터 암호화 과정

### 1. Store 초기화 및 Store Key 생성

```
Passphrase (또는 Raw Key)
    ↓
Argon2i KDF (salt 사용)
    ↓
Store Key (32 bytes, ChaCha20Poly1305 키)
```

**Store Key 생성 파라미터:**
- KDF: Argon2i
- Time Cost: 6 (moderate) 또는 4 (interactive)
- Memory Cost: 131072 (128MB, moderate) 또는 32768 (32MB, interactive)
- Parallelism: argon2 라이브러리 기본값 사용 (명시적으로 설정되지 않음)
- Hash Length: 32 bytes

### 2. Profile 생성 및 Profile Key 암호화

각 profile이 생성될 때:

1. **Profile Key 생성**
   - 6개의 랜덤 키 생성 (category_key, name_key, item_hmac_key, tag_name_key, tag_value_key, tags_hmac_key)
   - CBOR 형식으로 직렬화

2. **Profile Key 암호화**
   ```
   Profile Key (CBOR)
       ↓
   ChaCha20Poly1305 (Store Key 사용)
       ↓
   [Nonce (12 bytes)][Ciphertext + Tag (16 bytes)]
       ↓
   profiles 테이블에 저장
   ```

### 3. Item 데이터 암호화

Item은 다음 필드로 구성됩니다:
- **Category**: 검색 가능한 암호화 (searchable encryption)
- **Name**: 검색 가능한 암호화 (searchable encryption)
- **Value**: 파생 키로 암호화 (random nonce)
- **Tags**: 검색 가능한 암호화 (searchable encryption)

#### 3.1 Category 암호화

```
Plaintext Category
    ↓
HMAC-SHA-256(Plaintext Category, item_hmac_key)
    ↓
Nonce = HMAC 결과의 첫 12 bytes
    ↓
ChaCha20Poly1305(Category, category_key, Nonce)
    ↓
[Nonce (12 bytes)][Ciphertext + Tag (16 bytes)]
```

**특징:**
- 동일한 category는 항상 동일한 암호문 생성 (deterministic)
- 검색 가능 (같은 category로 검색 가능)

#### 3.2 Name 암호화

```
Plaintext Name
    ↓
HMAC-SHA-256(Plaintext Name, item_hmac_key)
    ↓
Nonce = HMAC 결과의 첫 12 bytes
    ↓
ChaCha20Poly1305(Name, name_key, Nonce)
    ↓
[Nonce (12 bytes)][Ciphertext + Tag (16 bytes)]
```

**특징:**
- 동일한 name은 항상 동일한 암호문 생성 (deterministic)
- 검색 가능

#### 3.3 Value 암호화

```
Plaintext Value
    ↓
Value Key 파생:
HMAC-SHA-256(
    u_int32(len(category)) || category ||
    u_int32(len(name)) || name,
    item_hmac_key
)
    ↓
Value Key = HMAC 결과 (32 bytes)
    ↓
Random Nonce 생성 (12 bytes)
    ↓
ChaCha20Poly1305(Value, Value Key, Random Nonce)
    ↓
[Nonce (12 bytes)][Ciphertext + Tag (16 bytes)]
```

**특징:**
- 매번 다른 nonce 사용 (non-deterministic)
- Category와 Name 조합에 따라 다른 키 사용
- 검색 불가능 (보안 강화)

#### 3.4 Tags 암호화

**Tag Name:**
```
Plaintext Tag Name
    ↓
HMAC-SHA-256(Plaintext Tag Name, tags_hmac_key)
    ↓
Nonce = HMAC 결과의 첫 12 bytes
    ↓
ChaCha20Poly1305(Tag Name, tag_name_key, Nonce)
```

**Tag Value (암호화된 태그인 경우):**
```
Plaintext Tag Value
    ↓
HMAC-SHA-256(Plaintext Tag Value, tags_hmac_key)
    ↓
Nonce = HMAC 결과의 첫 12 bytes
    ↓
ChaCha20Poly1305(Tag Value, tag_value_key, Nonce)
```

**특징:**
- Tag Name과 Tag Value 모두 검색 가능
- Plaintext 태그도 지원 (plaintext 플래그 사용)

## 데이터베이스 스키마

### config 테이블

| 컬럼명 | 타입 | 설명 |
|--------|------|------|
| `name` | string | 설정 키 (PK, 예: `default_profile`, `key`, `version`) |
| `value` | string | 설정 값 (`default_profile`의 경우 기본 profile 이름, `key`의 경우 Store Key 메타데이터, `version`의 경우 스키마 버전) |

### profiles 테이블

| 컬럼명 | 타입 | 설명 |
|--------|------|------|
| `id` | BIGSERIAL/INTEGER | Profile ID (PK) |
| `name` | TEXT | Profile 이름 (UNIQUE) |
| `profile_key` | BYTEA/BLOB | 암호화된 Profile Key |
| `reference` | string | 참조 정보 (선택적) |

### items 테이블

| 컬럼명 | 타입 | 설명 |
|--------|------|------|
| `id` | BIGSERIAL/INTEGER | Item ID (PK) |
| `profile_id` | BIGINT | Profile ID (FK) |
| `kind` | SMALLINT | KMS (1) 또는 Item (2) |
| `category` | BYTEA | 암호화된 Category |
| `name` | BYTEA | 암호화된 Name |
| `value` | BYTEA | 암호화된 Value |
| `expiry` | TIMESTAMP/DATETIME | 만료 시간 |

### items_tags 테이블

| 컬럼명 | 타입 | 설명 |
|--------|------|------|
| `id` | BIGSERIAL/INTEGER | Tag ID (PK) |
| `item_id` | BIGINT | Item ID (FK) |
| `name` | BYTEA | 암호화된 Tag Name |
| `value` | BYTEA | 암호화된 Tag Value |
| `plaintext` | BOOLEAN/SMALLINT | 평문 여부 |

## 복호화 과정

### 1. Store Key 복원
```
Passphrase + Salt
    ↓
Argon2i KDF
    ↓
Store Key
```

### 2. Profile Key 복호화
```
암호화된 Profile Key (profiles 테이블)
    ↓
Store Key로 복호화
    ↓
Profile Key (CBOR)
    ↓
CBOR 디코딩
    ↓
6개의 키 추출
```

### 3. Item 데이터 복호화

#### Category 복호화
```
암호화된 Category
    ↓
Nonce 추출 (첫 12 bytes)
    ↓
ChaCha20Poly1305 복호화 (category_key 사용)
    ↓
Plaintext Category
```

#### Name 복호화
```
암호화된 Name
    ↓
Nonce 추출 (첫 12 bytes)
    ↓
ChaCha20Poly1305 복호화 (name_key 사용)
    ↓
Plaintext Name
```

#### Value 복호화
```
암호화된 Value
    ↓
Nonce 추출 (첫 12 bytes)
    ↓
Category와 Name으로 Value Key 파생
    ↓
ChaCha20Poly1305 복호화 (Value Key 사용)
    ↓
Plaintext Value
```

## 보안 특징

1. **키 격리**: 각 profile은 독립적인 키를 가지므로, 한 profile의 데이터는 다른 profile의 키로 복호화할 수 없습니다.

2. **검색 가능한 암호화**: Category, Name, Tags는 deterministic 암호화를 사용하여 검색이 가능하지만, Value는 non-deterministic으로 더 높은 보안을 제공합니다.

3. **키 파생**: Value Key는 category와 name의 조합으로부터 파생되므로, 같은 category와 name을 가진 항목들도 서로 다른 Value Key를 사용할 수 있습니다.

4. **Store Key 보호**: Store Key는 데이터베이스에 저장되지 않으며, 메모리에서만 사용됩니다. Passphrase 기반 KDF를 사용하면 brute force 공격에 대한 보호가 제공됩니다.

## 다이어그램

### 전체 키 계층 구조 및 암호화 흐름

```mermaid
graph TB
    subgraph "Store 초기화"
        Passphrase[Passphrase 또는 Raw Key]
        Salt[Salt<br/>16 bytes]
        Passphrase -->|Argon2i KDF| StoreKey[Store Key<br/>32 bytes<br/>ChaCha20Poly1305]
        Salt -->|Argon2i KDF| StoreKey
    end

    subgraph "Profile 1"
        PK1[Profile Key 1<br/>CBOR 형식]
        PK1 -->|Store Key로 암호화| EPK1[암호화된 Profile Key 1<br/>profiles 테이블]
        PK1 -->|추출| Keys1[6개 키:<br/>- category_key<br/>- name_key<br/>- item_hmac_key<br/>- tag_name_key<br/>- tag_value_key<br/>- tags_hmac_key]
    end

    subgraph "Profile 2"
        PK2[Profile Key 2<br/>CBOR 형식]
        PK2 -->|Store Key로 암호화| EPK2[암호화된 Profile Key 2<br/>profiles 테이블]
        PK2 -->|추출| Keys2[6개 키:<br/>- category_key<br/>- name_key<br/>- item_hmac_key<br/>- tag_name_key<br/>- tag_value_key<br/>- tags_hmac_key]
    end

    subgraph "Profile N"
        PKN[Profile Key N<br/>CBOR 형식]
        PKN -->|Store Key로 암호화| EPKN[암호화된 Profile Key N<br/>profiles 테이블]
        PKN -->|추출| KeysN[6개 키]
    end

    StoreKey -->|복호화| PK1
    StoreKey -->|복호화| PK2
    StoreKey -->|복호화| PKN

    style StoreKey fill:#ff9999
    style PK1 fill:#99ccff
    style PK2 fill:#99ccff
    style PKN fill:#99ccff
    style EPK1 fill:#ffcc99
    style EPK2 fill:#ffcc99
    style EPKN fill:#ffcc99
```

### Item 데이터 암호화 과정

```mermaid
graph LR
    subgraph "입력 데이터"
        PC[Plaintext Category]
        PN[Plaintext Name]
        PV[Plaintext Value]
        PT[Plaintext Tags]
    end

    subgraph "Profile Key에서 추출된 키"
        CK[category_key<br/>ick]
        NK[name_key<br/>ink]
        IHK[item_hmac_key<br/>ihk]
        TNK[tag_name_key<br/>tnk]
        TVK[tag_value_key<br/>tvk]
        THK[tags_hmac_key<br/>thk]
    end

    subgraph "Category 암호화"
        PC -->|HMAC-SHA-256| HMC[HMAC 결과]
        HMC -->|첫 12 bytes| NC[Nonce]
        PC -->|ChaCha20Poly1305| EC[암호화된 Category<br/>Nonce + Ciphertext + Tag]
        CK -->|암호화 키| EC
        NC -->|Nonce| EC
    end

    subgraph "Name 암호화"
        PN -->|HMAC-SHA-256| HMN[HMAC 결과]
        HMN -->|첫 12 bytes| NN[Nonce]
        PN -->|ChaCha20Poly1305| EN[암호화된 Name<br/>Nonce + Ciphertext + Tag]
        NK -->|암호화 키| EN
        NN -->|Nonce| EN
        IHK -->|HMAC 키| HMN
    end

    subgraph "Value 암호화"
        PV -->|ChaCha20Poly1305| EV[암호화된 Value<br/>Random Nonce + Ciphertext + Tag]
        PC -->|Value Key 파생| VKD[HMAC-SHA-256<br/>len+category+len+name]
        PN -->|Value Key 파생| VKD
        IHK -->|HMAC 키| VKD
        VKD -->|32 bytes| VK[Value Key]
        VK -->|암호화 키| EV
        RN[Random Nonce<br/>12 bytes] -->|Nonce| EV
    end

    subgraph "Tags 암호화"
        PT -->|Tag Name| TN[Tag Name]
        PT -->|Tag Value| TV[Tag Value]
        TN -->|HMAC-SHA-256| HMTN[HMAC 결과]
        TV -->|HMAC-SHA-256| HMTV[HMAC 결과]
        HMTN -->|첫 12 bytes| NTN[Nonce]
        HMTV -->|첫 12 bytes| NTV[Nonce]
        TN -->|ChaCha20Poly1305| ETN[암호화된 Tag Name]
        TV -->|ChaCha20Poly1305| ETV[암호화된 Tag Value]
        TNK -->|암호화 키| ETN
        TVK -->|암호화 키| ETV
        THK -->|HMAC 키| HMTN
        THK -->|HMAC 키| HMTV
        NTN -->|Nonce| ETN
        NTV -->|Nonce| ETV
    end

    IHK -->|HMAC 키| HMC

    style EC fill:#ffcc99
    style EN fill:#ffcc99
    style EV fill:#ffcc99
    style ETN fill:#ffcc99
    style ETV fill:#ffcc99
    style VK fill:#99ff99
```

### 데이터베이스 저장 구조

```mermaid
erDiagram
    CONFIG ||--o{ PROFILES : "참조"
    PROFILES ||--o{ ITEMS : "profile_id"
    ITEMS ||--o{ ITEMS_TAGS : "item_id"

    CONFIG {
        string name PK
        string value
    }

    PROFILES {
        bigint id PK
        string name UK
        bytea profile_key "암호화된 Profile Key"
        string reference
    }

    ITEMS {
        bigint id PK
        bigint profile_id FK
        smallint kind
        bytea category "암호화된 Category"
        bytea name "암호화된 Name"
        bytea value "암호화된 Value"
        timestamp expiry
    }

    ITEMS_TAGS {
        bigint id PK
        bigint item_id FK
        bytea name "암호화된 Tag Name"
        bytea value "암호화된 Tag Value"
        boolean plaintext
    }
```

### 복호화 과정

```mermaid
sequenceDiagram
    participant App as 애플리케이션
    participant Store as Askar Store
    participant DB as 데이터베이스
    participant Cache as KeyCache

    App->>Store: Item 조회 요청 (profile, category, name)
    Store->>Cache: Profile Key 조회
    alt Cache에 없음
        Cache->>DB: profiles 테이블에서 암호화된 Profile Key 조회
        DB-->>Cache: 암호화된 Profile Key
        Cache->>Cache: Store Key로 복호화
        Cache->>Cache: CBOR 디코딩하여 6개 키 추출
        Cache->>Cache: 메모리 캐시에 저장
    end
    Cache-->>Store: Profile Key (6개 키 포함)
    
    Store->>DB: items 테이블에서 암호화된 데이터 조회
    DB-->>Store: 암호화된 Category, Name, Value, Tags
    
    Store->>Store: Category 복호화<br/>(category_key 사용)
    Store->>Store: Name 복호화<br/>(name_key 사용)
    Store->>Store: Value Key 파생<br/>(category + name + item_hmac_key)
    Store->>Store: Value 복호화<br/>(Value Key 사용)
    Store->>Store: Tags 복호화<br/>(tag_name_key, tag_value_key 사용)
    
    Store-->>App: 복호화된 Item 데이터
```

### Multitenancy 구조

```mermaid
graph TB
    subgraph "Askar Store"
        SK[Store Key<br/>단일 인스턴스]
        
        subgraph "Profile 1"
            PK1[Profile Key 1]
            I1[Items 1]
            T1[Tags 1]
            PK1 -->|암호화| I1
            PK1 -->|암호화| T1
        end
        
        subgraph "Profile 2"
            PK2[Profile Key 2]
            I2[Items 2]
            T2[Tags 2]
            PK2 -->|암호화| I2
            PK2 -->|암호화| T2
        end
        
        subgraph "Profile N"
            PKN[Profile Key N]
            IN[Items N]
            TN[Tags N]
            PKN -->|암호화| IN
            PKN -->|암호화| TN
        end
        
        SK -->|암호화| PK1
        SK -->|암호화| PK2
        SK -->|암호화| PKN
    end
    
    subgraph "데이터베이스"
        DB[(profiles 테이블<br/>items 테이블<br/>items_tags 테이블)]
        PK1 -.->|저장| DB
        PK2 -.->|저장| DB
        PKN -.->|저장| DB
        I1 -.->|저장| DB
        I2 -.->|저장| DB
        IN -.->|저장| DB
    end

    style SK fill:#ff9999
    style PK1 fill:#99ccff
    style PK2 fill:#99ccff
    style PKN fill:#99ccff
    style DB fill:#ffcc99
```

### 검색 가능한 암호화 vs 일반 암호화

```mermaid
graph TB
    subgraph "검색 가능한 암호화<br/>(Searchable Encryption)"
        SE1[Plaintext Category/Name/Tag]
        SE1 -->|HMAC-SHA-256| SE2[HMAC 결과]
        SE2 -->|첫 12 bytes| SE3[Deterministic Nonce]
        SE1 -->|ChaCha20Poly1305| SE4[암호화된 데이터]
        SE3 -->|Nonce| SE4
        SE4 -->|특징| SE5[동일한 입력 →<br/>동일한 출력<br/>검색 가능]
    end

    subgraph "일반 암호화<br/>(Value)"
        VE1[Plaintext Value]
        VE1 -->|Value Key 파생| VE2[Category + Name 기반]
        VE2 -->|HMAC-SHA-256| VE3[Value Key]
        VE1 -->|ChaCha20Poly1305| VE4[암호화된 Value]
        VE5[Random Nonce<br/>12 bytes] -->|Nonce| VE4
        VE3 -->|암호화 키| VE4
        VE4 -->|특징| VE6[매번 다른 Nonce<br/>검색 불가능<br/>더 높은 보안]
    end

    style SE5 fill:#99ff99
    style VE6 fill:#ff9999
```

