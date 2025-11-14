# CMWallet

## Overview

CMWallet 은 Android Credential Manager 생태계에서 Credential Provider 역할을 하는 앱

## Architecture

![CMWallet Architecture](files/cmwallet-architecture.png "CMWallet Architecture")

### Key Features

* Digital Payment Credentials (DPC) 보관: 사용자의 결제 카드 정보를 디지털 자격 증명서로 보관
* Credential Manager 통합: Android 시스템의 Credential Manager API 를 통해 다른 앱에 자격 증명 제공
* 사용자 승인 UI: 결제 승인시 거래 세부 정보를 표시 하고 사용자 확인 요청
* 암호화 서명: 기기의 보안 요소를 사용하여 거래에 암호화 서명 생성

### 왜 별도 앱인가?
* 역할 분리: Shopping Agent 는 쇼핑 경험에 집중, CMWallet 은 결제 자격 증명 관리 역할에 집중
* 보안 격리: 민감한 결제 정보를 별도 앱에서 관리 하여 보안 강화
* 재 사용성: 하나의 CMWallet 이 여러 쇼핑 앱에서 사용 가능
* 표준 준수: Android Credential Manager 의 표준 아키텍처 패턴 준수

### 필수 요구사항

* CMWallet 은 Shopping Agent 와 동일 기기에 설치 되어야 함.


## Integration Flow with Android Credential Manager API

CMWallet integrates with the Android Credential Manager API to provide secure credential management.

```{mermaid}
sequenceDiagram
    participant SA as Shopping Assistant
    participant ACM as Android Credential Manager API
    participant CMW as CM Wallet (Credential Provider)

    SA->>ACM: (1) credentialManager.getCredential(request)
    ACM->>ACM: (2) Find appropriate Credential Provider
    ACM->>CMW: (3) Invoke CM Wallet
    CMW->>CMW: (4) Show UI & get user approval
    CMW->>ACM: (5) Generate signed token
    ACM->>SA: (6) Return token
```

## Features

### 1. Credential Storage

Description of credential storage...

### 2. Authentication

Description of authentication...

### 3. Security

Description of security features...

## Configuration

### Setup

```kotlin
// Configuration example
```

### Initialization

```kotlin
// Initialization example
```

## Usage Examples

### Basic Example

```kotlin
// Basic usage example
```

### Advanced Example

```kotlin
// Advanced usage example
```

## Best Practices

- Best practice 1
- Best practice 2
- Best practice 3

## References

- [Official Documentation](https://example.com)
- [API Reference](https://example.com/api)

