# Digital Payment Credential 시나리오

## Introduction

Sample 은 3가지 시나리오를 제공함

* Cards - 기존 신용카드 인프라 사용 시나리오
* x402 - HTTP 402 Code 확장 시나리오
* DPC - Digital Payment Credential 연계 시나리오

### Cards 시나리오

전통적인 신용/직불 카드 결제 인프라를 AI 에이전트가 활용하는 시나리오입니다.

#### 개요

Cards 시나리오는 기존의 **카드 결제 네트워크**(Visa, Mastercard 등)를 통해 AI 에이전트가 자동으로 결제를 수행하는 방식입니다. AP2의 세 가지 시나리오 중 **가장 현실적이고 즉시 도입 가능한** 접근 방식으로, 새로운 HTTP 기반 프로토콜(x402)이나 디지털 credential(DPC)과 달리 **이미 검증된 기존 카드 결제 인프라**를 그대로 활용하여 빠른 도입과 높은 호환성을 제공합니다.

#### 동작 방식

```{mermaid}
sequenceDiagram
    participant Agent as AI Agent
    participant Merchant as Merchant System
    participant Gateway as Payment Gateway
    participant Network as Card Network
    participant Bank as Issuing Bank
    
    Agent->>Merchant: (1) 구매 요청
    Merchant->>Gateway: (2) 결제 요청
    Gateway->>Network: (3) 승인 요청 전송
    Network->>Bank: (4) 사용자 계좌 확인
    Bank-->>Network: (5) 승인/거절 응답
    Network-->>Gateway: (6) 응답 전달
    Gateway-->>Merchant: (7) 결제 결과
    Merchant-->>Agent: (8) 구매 완료 확인
```

#### 주요 특징

1. **기존 인프라 활용**: Visa, Mastercard, AMEX 등 전 세계적으로 구축된 카드 네트워크 사용
2. **즉시 적용 가능**: 새로운 표준이나 프로토콜 도입 없이 기존 시스템과 호환
3. **높은 신뢰성**: 수십 년간 검증된 보안 및 사기 방지 시스템
4. **PCI DSS 준수**: 카드 산업 데이터 보안 표준을 따르는 안전한 처리
5. **광범위한 수용성**: 대부분의 가맹점에서 이미 지원하는 결제 수단

#### AI 에이전트 통합 방식

**1. 토큰화된 카드 정보**
```kotlin
// 에이전트가 안전하게 저장된 토큰화된 카드 정보 사용
val paymentToken = secureStorage.getPaymentToken()
val transaction = Transaction(
    amount = cartTotal,
    currency = "USD",
    paymentMethod = paymentToken,
    merchantId = merchantInfo.id
)
```

**2. 자동 승인 로직**
- 사전 설정된 지출 한도 내에서 자동 승인
- 한도 초과 시 사용자에게 확인 요청
- 이상 거래 패턴 감지 시 거래 중단

**3. 멀티 카드 관리**
- 여러 카드 중 최적의 카드 자동 선택 (포인트, 할인 등)
- 카드 잔액 및 한도 실시간 확인
- 거절 시 다른 카드로 자동 재시도

#### 사용 사례

- **구독 서비스 자동 결제**: AI 에이전트가 필요한 서비스를 자동으로 구독하고 결제
- **일상 구매**: 온라인 쇼핑에서 에이전트가 가격 비교 후 자동 구매
- **B2B 거래**: 기업용 카드를 통한 자동 발주 및 결제
- **동적 예산 관리**: 설정된 예산 범위 내에서 최적의 구매 결정

#### 다른 시나리오와의 비교

| 특징 | Cards | x402 | DPC |
|------|-------|------|-----|
| **기반 기술** | 카드 네트워크 | HTTP 402 | Digital Credentials |
| **도입 난이도** | 낮음 (기존 인프라) | 높음 (신규 표준) | 중간 (새로운 API) |
| **적용 범위** | 전통적 커머스 | 웹 리소스/API | 모바일 중심 |
| **사용자 개입** | 초기 설정 후 없음 | 거의 없음 | 승인 시 필요 |
| **보안 모델** | PCI DSS | 토큰 기반 | Cryptographic Proof |
| **주요 장점** | 즉시 사용 가능 | M2M 최적화 | 향후 표준 |

#### 한계점 및 고려사항

- **수수료**: 카드 네트워크 수수료 (보통 2-3%) 존재
- **보안 요구사항**: PCI DSS 준수를 위한 추가 보안 조치 필요
- **지역별 제약**: 일부 국가에서는 카드 사용 제한 또는 높은 수수료
- **개인정보**: 카드 정보 저장 및 관리에 대한 규제 준수 필요

### x402 시나리오

HTTP 402 "Payment Required" 상태 코드를 활용한 AI 에이전트 간 자동 결제 시나리오입니다.

#### 개요

x402는 1997년 HTTP/1.1 표준에 정의되었지만 거의 사용되지 않았던 **HTTP 402 상태 코드**를 현대적인 AI 에이전트 생태계에 맞게 재해석한 프로토콜입니다. AP2 샘플의 x402 시나리오는 이 개념을 구현한 것으로, AI 에이전트들이 웹 리소스나 서비스에 접근할 때 자동으로 결제를 처리할 수 있도록 합니다.

#### 동작 방식

```{mermaid}
sequenceDiagram
    participant Agent as AI Agent
    participant Server as Resource Server
    participant Payment as Payment Processor
    
    Agent->>Server: (1) 리소스 요청
    Server-->>Agent: (2) HTTP 402 Payment Required + 결제 정보
    Agent->>Payment: (3) 자동 결제 처리
    Payment-->>Agent: (4) 결제 증명 (토큰)
    Agent->>Server: (5) 결제 증명과 함께 재요청
    Server-->>Agent: (6) 리소스 제공
```

#### 주요 특징

1. **AI 에이전트 친화적**: 사람의 개입 없이 에이전트가 자율적으로 결제 결정
2. **프로토콜 독립적**: 특정 결제 수단이나 통화에 종속되지 않음
3. **표준 기반**: HTTP 표준을 확장하여 웹 생태계와 자연스럽게 통합
4. **마이크로 페이먼트**: 소액 결제에 최적화 (거의 무료에 가까운 수수료)
5. **중개자 불필요**: 판매자와 구매자 간 직접 거래 조건 설정 가능

#### 사용 사례

- **AI 에이전트 모니터링 서비스**: 에이전트의 활동 추적 및 성능 분석에 대한 결제
- **프리미엄 API 접근**: 유료 API나 데이터셋에 대한 자동 결제
- **컴퓨팅 리소스**: 필요한 만큼만 사용하고 즉시 결제하는 온디맨드 서비스
- **콘텐츠 마이크로 페이먼트**: 뉴스 기사, 연구 논문 등 개별 콘텐츠에 대한 소액 결제

#### AP2와의 관계

x402는 AP2(Agent to Payment) 프로토콜의 실험적 컨셉 중 하나로, HTTP 레벨에서 결제를 자연스럽게 통합하는 방법을 보여줍니다. Cards 시나리오가 기존 인프라를 활용하고, DPC 시나리오가 모바일 환경에서의 구현에 초점을 맞춘 반면, x402는 **웹 서비스와 AI 에이전트 간의 M2M(Machine-to-Machine) 결제**에 중점을 두며, 장기적으로는 인터넷 네이티브 결제 시스템의 표준이 될 가능성을 탐색합니다.

### DPC 시나리오

디지털 자격증명(Digital Credentials)을 활용한 차세대 모바일 결제 시나리오입니다.

#### 개요

DPC(Digital Payment Credential) 시나리오는 **암호학적으로 검증 가능한 디지털 자격증명**을 결제 수단으로 활용하는 혁신적인 접근 방식입니다. x402가 HTTP 프로토콜에, Cards가 기존 카드 네트워크에 기반한다면, DPC는 **EUDI Wallet과 같은 디지털 신원 표준**을 결제 영역으로 확장한 것입니다.

Android Credential Manager API를 통해 CMWallet과 상호작용하며, 사용자는 암호학적으로 서명된 증명(cryptographic proof)을 제시하여 결제를 완료합니다. 이는 단순히 카드 정보를 전달하는 것이 아니라, **사용자가 특정 자격증명의 소유자임을 증명**하는 방식입니다.

#### 동작 방식

```{mermaid}
sequenceDiagram
    participant SA as Shopping Assistant
    participant MA as Merchant Agent
    participant ACM as Android Credential Manager API
    participant CMW as CM Wallet

    SA->>MA: (1) 장바구니 확정 요청 (네트워크)
    MA-->>SA: (2) Cart Mandate 반환
    SA->>ACM: (3) DPC 요청 생성 (Credential Manager API 호출)
    ACM->>CMW: (4) CM Wallet 호출<br/>UI 표시 & 서명 생성
    CMW-->>ACM: (5) 서명된 토큰 반환
    ACM-->>SA: (5) 토큰 수신
    SA->>MA: (6) 토큰 전송 (네트워크)
    MA-->>MA: 토큰 검증 및 결제 처리
```

#### 주요 특징

1. **암호학적 보안**: 공개키 암호화 기반의 서명으로 위변조 불가능
2. **선택적 공개 (Selective Disclosure)**: 필요한 정보만 선택적으로 공개 (예: 카드 번호, 소유자 이름만)
3. **사용자 통제**: 모든 거래에서 사용자가 명시적으로 승인 (UI를 통한 확인)
4. **표준 기반**: OpenID4VP, ISO 18013-5 mDOC, DCQL 등 국제 표준 준수
5. **프라이버시 보호**: 최소한의 정보만 공유하며, 추적 방지 메커니즘 내장
6. **오프라인 검증 가능**: 서명된 자격증명은 오프라인에서도 검증 가능
7. **플랫폼 네이티브**: Android Credential Manager API와 긴밀히 통합

#### 사용 사례

- **프라이버시 중심 결제**: 최소한의 개인정보만 공개하면서 결제하는 고급 사용자
- **규제 준수 환경**: GDPR, EUDI Wallet 등 엄격한 프라이버시 규제를 따라야 하는 EU 시장
- **고가 거래**: 강력한 인증이 필요한 고액 결제 (예: 부동산, 명품, 귀금속)
- **신원 연계 결제**: 결제와 동시에 연령 확인이나 자격 증명이 필요한 경우 (예: 주류, 담배)
- **B2B 거래**: 기업 자격증명을 활용한 기업 간 거래
- **크로스보더 거래**: 국제 표준 기반으로 국경을 넘는 거래에 적합

#### 기술 상세

##### CMWallet 구조

CMWallet은 Digital Credential을 관리하는 전용 앱으로, 샘플에서는 2개의 가상 신용카드를 제공합니다:
- 사용자는 결제 시 복수의 카드 중 선택 가능
- 각 카드는 ISO 18013-5 mDOC 형식으로 저장
- ES256 알고리즘으로 서명되어 위변조 방지

##### Android Credential Manager API 통합

```kotlin
// Credential Manager를 통한 DPC 요청
val credentialManager = CredentialManager.create(context)
val result = credentialManager.getCredential(
    request = GetCredentialRequest(
        credentialOptions = listOf(dpcRequest)
    )
)
```

## DPC 구현 분석 (DpcHelper.kt)

이 코드는 DPC (Digital Payment Credential) 요청을 생성함. 주요 기능은 쇼핑 카트 정보를 받아 OID4VP (OpenID 4 Verifiable Presentation) 프로토콜 기반의 인증 요청 생성.

## EUID Wallet (EU Digital Identity Wallet) 표준 연관성

DPC 시나리오는 EUID Wallet 표준과 여러 측면에서 밀접하게 연관 있음

### OID4VP (OpenID for Verifiable Presentaiton) 프로토콜 사용

```{code-block} kotlin
:caption: DpcHelper.kt:110~120

val dcRequest =
  Request(
    responseType = "vp_token",
    responseMode = "dc_api",
    nonce = nonce,
    dcqlQuery = dcqlQuery,
    transactionData = listOf(encodedTransactionData),
    clientMetadata = clientMetadata,
  )

val dpcRequest = DpcRequest(protocol = "openid4vp-v1-unsigned", request = dcRequest)
```

* EUDI Wallet 표준의 핵심: EUDI Wallet 은 OpenID4VP 를 주요 프로토콜로 사용함.
* Protocol = “openid4vp-v1-unsigned” - EUDI Wallet 도 동일한 프로토콜을 사용하여 Verifiable Presentation 을 요청
* responseType = “vp_token”

### ISO/IEC 18013-5 mode 형식 지원

```{code-block} kotlin
:caption: DpcHelper.kt:85~91

val credentialQuery =
    CredentialQuery(
        id = credId,
        format = mdocIdentifier,
        meta = Meta(doctypeValue = "com.emvco.payment_card"),
        claims = claims,
)
```

```{code-block} kotlin
:caption: DpcHelper.kt:96~100

val mdocFormatsSupported =
    MdocFormatsSupported(
        issuerauthAlgValues = listOf(-7), // ES256
        deviceauthAlgValues = listOf(-7),
    )
```

* EUID Wallet 의 주요 형식: EUID Wallet 은 ISO mdoc (mDL - Mobile Driver’s License) 형식을 핵심 credential 형식으로 사용
* Format = mdocIdentifier (mso_mdoc) - EUID Wallet 에서도 동일한 mdoc 형식 사용
* ES256 알고리즘 - EUID Wallet 에서 권장하는 암호화 알고리즘

### DCQL (Digital Credentials Query Language) 사용

```{code-block} kotlin
:caption: DpcHelper.kt:78~93

  // Build the DCQL query to request specific credential claims.
  val claims =
    listOf(
      Claim(path = listOf("com.emvco.payment_card.1", "card_number")),
      Claim(path = listOf("com.emvco.payment_card.1", "holder_name")),
    )

  val credentialQuery =
    CredentialQuery(
      id = credId,
      format = mdocIdentifier,
      meta = Meta(doctypeValue = "com.emvco.payment_card"),
      claims = claims,
    )

  val dcqlQuery = DcqlQuery(credentials = listOf(credentialQuery))
```

* 선택적 공개 (Selective Disclosure) - EUDI Wallet 의 핵심 원칙
* DCQL 을 사용하여 필요한 특정 claim 만 요청 (카드 번호, 소유자 이름 만)
* EUDI Wallet 도 동일한 방식으로 사용자가 공개할 정보를 선택할 수 있음

### Android Credential Manager API 통합

```{code-block} kotlin
:caption: DpcHelpper.kt:67~76

  // Build transaction_data payload.
  val transactionData =
    TransactionData(
      type = "payment_card",
      credentialIds = listOf(credId),
      transactionDataHashesAlg = listOf("sha-256"),
      merchantName = merchantName,
      amount = "US ${String.format("%.2f", totalValue)}",
      additionalInfo = json.encodeToString(additionalInfo), // Serialize the inner object
    )
```

* Android Credential Manager API 는 EUDI Wallet 의 구현 플랫폼 중 하나임
* Transaction Data 에 대한 서명 - EUDI Wallet 에서도 중요한 보안 메커니즘
* transactionDataHashesAlg = “sha-256” - 거래 무결성 보장

```{code-block} kotkin
:caption: DpcHelper.kt:59~65
  val additionalInfo =
    AdditionalInfo(
      title = "Please confirm your purchase details...",
      tableHeader = listOf("Name", "Qty", "Price", "Total"),
      tableRows = tableRows,
      footer = footerText,
    )
```

* EUID Wallet 의 핵심 원칙: 사용자가 공유하는 정보를 명확히 보고 동의해야 함
* 구매 세부사항을 표시하여 사용자가 서명하는 내용을 정확히 이해할 수 있게 함

### DPC vs EUDI Wallet 차이점

* **용도**: DPC는 결제에 특화 (payment_card), EUDI Wallet은 신원 증명이 주요 목적
* **doctype**: com.emvco.payment_card vs EUDI Wallet의 eu.europa.ec.eudi.pid.1 (Personal ID)
* **서명**: 현재는 unsigned 버전을 사용하지만, EUDI Wallet은 강력한 암호화 서명 요구

#### 다른 시나리오와의 비교

| 특징 | Cards | x402 | DPC |
|------|-------|------|-----|
| **기반 기술** | 카드 네트워크 | HTTP 402 | Digital Credentials |
| **보안 수준** | PCI DSS | 토큰 기반 | 암호학적 증명 |
| **프라이버시** | 낮음 | 중간 | 매우 높음 (Selective Disclosure) |
| **사용자 경험** | 익숙함 | 투명함 | 명시적 승인 필요 |
| **도입 장벽** | 낮음 | 높음 | 중간 |
| **미래 전망** | 안정적 | 실험적 | 성장 가능성 높음 |
| **국제 표준** | 확립됨 | 제안 단계 | 표준화 진행 중 (EUDI) |
| **거래 비용** | 2-3% 수수료 | 거의 없음 | 매우 낮음 |
| **오프라인 지원** | 제한적 | 불가능 | 가능 (서명 검증) |

#### 한계점 및 고려사항

**현재의 제약사항:**
- **플랫폼 한정**: 현재 Android에만 구현, iOS 지원 제한적
- **생태계 미성숙**: CMWallet 등 credential provider가 아직 초기 단계
- **사용자 교육**: 새로운 개념으로 사용자 이해와 신뢰 구축 필요
- **표준화 진행 중**: OpenID4VP, mDOC 등이 아직 완전히 확립되지 않음

**보안 및 개인정보:**
- **검증 메커니즘**: 현재 샘플은 unsigned 버전, 실제 운영에서는 완전한 서명 검증 필수
- **키 관리**: 사용자 기기에서의 안전한 키 저장 및 관리 중요
- **재생 공격 방지**: Nonce 및 타임스탬프 검증 필요
- **프라이버시 역설**: 강력한 인증이 익명성과 상충될 수 있음

**기술적 고려사항:**
- **성능**: 암호학적 연산으로 인한 지연 시간 (일반적으로 1-2초)
- **저장 공간**: mDOC 형식의 credential 저장에 필요한 공간
- **네트워크 의존성**: 초기 credential 발급 시 온라인 연결 필요
- **호환성**: 다양한 Android 버전 및 기기 지원 필요

**사업적 고려사항:**
- **가맹점 수용**: 새로운 결제 방식에 대한 가맹점 시스템 업그레이드 필요
- **규제 준수**: 각국의 금융 규제 및 데이터 보호법 준수
- **비즈니스 모델**: 기존 카드 네트워크와의 수익 모델 차이
- **사용자 인센티브**: 복잡한 새 시스템 도입을 정당화할 혜택 제공 필요

#### 결론

DPC는 EUDI Wallet과 동일한 기술 스택과 표준(OpenID4VP, ISO mdoc, DCQL)을 사용하여 결제 시나리오를 구현한 혁신적인 접근입니다. 

**세 시나리오의 위치:**
- **Cards**: 현재 - 즉시 도입 가능, 검증된 인프라
- **x402**: 실험 - 장기적 비전, 인터넷 네이티브 결제
- **DPC**: 성장 - 표준화 진행 중, 프라이버시 중심

DPC는 전통적인 카드 결제(Cards)의 편의성과 새로운 HTTP 기반 프로토콜(x402)의 유연성 사이에서 균형을 잡으며, **프라이버시와 보안을 최우선**으로 하는 차세대 결제 방식을 제시합니다. 특히 EU의 EUDI Wallet 같은 디지털 신원 이니셔티브와 자연스럽게 통합될 수 있어, **규제 준수와 사용자 통제가 중요한 미래 시장**에서 경쟁력을 가질 것으로 예상됩니다.

## Next Steps

### Verification of signed token

이 시나리오에서 Merchant Agent 는 Shopping Assistant 로 부터 받은 token 을 검증하지 않는다. 단지 signed token 이 포함되어 있는지 여부만 확인함. 부인 방지를 위해 추후 검증 방식이 보완 되어야 하며 이는 EUID Wallet 의 검증 방식을 차용할 것으로 예상됨.

### iOS 플랫폼 지원 및 상호운용성

현재 DPC 시나리오는 Android Credential Manager API를 기반으로 구현되어 있습니다. 진정한 크로스 플랫폼 상호운용성을 위해서는 iOS 지원이 필수적입니다.

#### Apple의 Digital Credentials API

Apple은 2024-2025년부터 **Digital Credentials API**를 도입하여 Android의 Credential Manager API와 유사한 기능을 제공하기 시작했습니다:

**현재 지원 현황:**
- ✅ ISO 18013-5 mDOC 형식 지원
- ✅ ISO 18013-7 Annex C 프로토콜 지원
- ✅ Apple Wallet 통합
- ❌ OpenID4VP 프로토콜 제한적 지원

#### 플랫폼 간 차이점

| 측면 | Android | iOS |
|------|---------|-----|
| **API** | Credential Manager API | Digital Credentials API + Apple Wallet |
| **프로토콜** | OpenID4VP 완전 지원 | ISO 18013-7 위주, OpenID4VP 제한적 |
| **생태계** | 오픈 (다양한 Provider) | 폐쇄적 (Apple Wallet 중심) |
| **유연성** | 높음 | 제한적 |

#### 상호운용성 확보 방안

EUDI Wallet의 접근 방식을 참고하여 다음과 같은 전략을 고려해야 합니다:

1. **플랫폼 추상화 레이어**: 각 OS의 네이티브 API를 감싸는 공통 인터페이스 구현
2. **공통 표준 활용**: OpenID4VCI, SD-JWT와 같은 플랫폼 독립적 프로토콜 사용
3. **iOS 네이티브 구현**: 
   - Swift로 구현
   - Apple App Attest를 통한 무결성 보장
   - IdentityDocumentServices 프레임워크 활용
4. **폴백 메커니즘**: OS 네이티브 지원이 부족할 경우 애플리케이션 레벨에서 구현

#### 향후 과제

- Apple의 Digital Credentials API에서 OpenID4VP 완전 지원 대기
- iOS용 CMWallet 구현
- 크로스 플랫폼 테스트 및 검증
- Apple의 제한적 생태계 내에서의 최적화

## References

- [Official Documentation](https://cloud.google.com/blog/products/ai-machine-learning/announcing-agents-to-payments-ap2-protocol)
- [GitHub Repository](https://github.com/google-agentic-commerce/AP2)
