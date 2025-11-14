# Hands On

## Introduction

Sample 은 3가지 시나리오를 제공함

* x402
* Cards
* DCP

### DCP 시나리오

Android 기반으로 되어 있으며 Android Credential Manager API 를 통해 CMWallet 과 상호작용 함

## DcpHelper.kt

이 코드는 DCP (Digital Payment Credential) 요청을 생성함. 주요 기능은 쇼핑 카트 정보를 받아 OID4VP (OpenID 4 Verifiable Presentation) 프로토콜 기반의 인증 요청 생성.

## EUID Wallet (EU Digital Identity Wallet) 표준 연관성

DCP 시나리오는 EUID Wallet 표준과 여러 측면에서 밀접하게 연관 있음

### OID4VP (OpenID for Verifiable Presentaiton) 프로토콜 사용

```{code-block} kotlin
:caption: DcpHelper.kt:110~120

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
:caption: DcpHelper.kt:85~91

val credentialQuery =
    CredentialQuery(
        id = credId,
        format = mdocIdentifier,
        meta = Meta(doctypeValue = "com.emvco.payment_card"),
        claims = claims,
)
```

```{code-block} kotlin
:caption: DcpHelper.kt:96~100

val mdocFormatsSupported =
    MdocFormatsSupported(
        issuerauthAlgValues = listOf(-7), // ES256
        deviceauthAlgValues = listOf(-7),
    )
```

## Next Steps

- Learn about [Advanced Topics](#)
- Explore [API Reference](#)
- Check out [Examples](#)

## References

- [Official Documentation](https://example.com)
- [GitHub Repository](https://github.com/example)
- [Community Forum](https://forum.example.com)

