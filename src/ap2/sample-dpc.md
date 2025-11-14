# Digital Payment Credential 시나리오

## Introduction

Sample 은 3가지 시나리오를 제공함

* x402 - HTTP 402 Code 확장 시나리오
* Cards - ?
* DPC - Digital Payment Credential 연계 시나리오

### x402 시나리오

### Cards 시나리오

### DPC 시나리오

Android 기반으로 되어 있으며 Android Credential Manager API 를 통해 CMWallet 과 상호작용 함. CMWallet 이란 Digital Credential 을 관리하는 전용 앱으로 샘플에서 제공하는 CMWallet 은 2개의 가상 신용카드를 가지고 있어 결제 확인시 사용자가 신용카드를 선택할 수 있음.

#### Digital Payment Credential 흐름

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


## DpcHelper.kt

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

### 주요 차이점

* 용도: 이 코드는 결제에 특화 (payment_card), EUDI Wallet은 신원 증명이 주요 목적
* doctype: com.emvco.payment_card vs EUDI Wallet의 eu.europa.ec.eudi.pid.1 (Personal ID)
* 서명: 현재는 unsigned 버전을 사용하지만, EUDI Wallet은 강력한 암호화 서명 요구

## 결론

이 코드는 EUDI Wallet과 동일한 기술 스택과 표준(OpenID4VP, ISO mdoc, DCQL)을 사용하여 결제 시나리오를 구현한 것입니다. EUDI Wallet 표준을 결제 분야에 적용한 좋은 예시로, 향후 EUDI Wallet과의 상호운용성을 고려한 설계라고 볼 수 있습니다.

## Next Steps

- Learn about [Advanced Topics](#)
- Explore [API Reference](#)
- Check out [Examples](#)

## References

- [Official Documentation](https://example.com)
- [GitHub Repository](https://github.com/example)
- [Community Forum](https://forum.example.com)

