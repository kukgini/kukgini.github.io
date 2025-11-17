# Cards 시나리오

This illustrates the complete flow of the Human Present Card Payment scenario using the AP2 framework with the A2A protocol.

## Sequence Diagram

```{mermaid}
sequenceDiagram
    participant User
    participant ShoppingAgent as Shopping Agent
    participant MerchantAgent as Merchant Agent
    participant CredentialsProvider as Credentials Provider
    participant PaymentProcessor as Payment Processor

    Note over User,PaymentProcessor: Phase 1: Product Discovery and Cart Creation

    User->>ShoppingAgent: I want to buy a coffee maker
    activate ShoppingAgent
    ShoppingAgent->>ShoppingAgent: Create IntentMandate
    Note over ShoppingAgent: IntentMandate: natural language description, merchants, SKUs

    ShoppingAgent->>MerchantAgent: A2A Message: Find products
    activate MerchantAgent
    Note over MerchantAgent: Validate shopping_agent_id
    MerchantAgent->>MerchantAgent: Search product catalog
    MerchantAgent->>MerchantAgent: Create CartMandate
    Note over MerchantAgent: CartMandate signed by merchant
    MerchantAgent-->>ShoppingAgent: CartMandate with products
    deactivate MerchantAgent

    ShoppingAgent-->>User: Display product options
    User->>ShoppingAgent: Select product
    ShoppingAgent->>ShoppingAgent: Store selected CartMandate

    Note over User,PaymentProcessor: Phase 2: Shipping Address Collection

    ShoppingAgent->>User: Request shipping address
    User->>ShoppingAgent: Provide shipping address
    ShoppingAgent->>ShoppingAgent: Store ContactAddress

    ShoppingAgent->>MerchantAgent: A2A Message: Update cart with shipping
    activate MerchantAgent
    MerchantAgent->>MerchantAgent: Update CartMandate with shipping
    MerchantAgent->>MerchantAgent: Re-sign CartMandate
    MerchantAgent-->>ShoppingAgent: Updated CartMandate
    deactivate MerchantAgent

    Note over User,PaymentProcessor: Phase 3: Payment Method Collection

    ShoppingAgent->>User: Link your Credentials Provider
    User->>ShoppingAgent: Confirm link

    ShoppingAgent->>CredentialsProvider: A2A Message: Get payment methods
    activate CredentialsProvider
    Note over CredentialsProvider: Filter by supported payment methods
    CredentialsProvider-->>ShoppingAgent: List of payment method aliases
    deactivate CredentialsProvider

    ShoppingAgent-->>User: Display payment methods
    User->>ShoppingAgent: Select payment method

    ShoppingAgent->>CredentialsProvider: A2A Message: Get payment credential token
    activate CredentialsProvider
    CredentialsProvider->>CredentialsProvider: Generate token for payment method
    Note over CredentialsProvider: Token includes DPAN card data
    CredentialsProvider-->>ShoppingAgent: Payment credential token
    deactivate CredentialsProvider

    Note over User,PaymentProcessor: Phase 4: Payment Mandate Creation and Signing

    ShoppingAgent->>ShoppingAgent: Create PaymentMandate
    Note over ShoppingAgent: PaymentMandate contains: cart details, payment method, shipping address

    ShoppingAgent-->>User: Display final purchase details
    Note over User: Merchant, item, price, shipping, tax, total
    User->>ShoppingAgent: Confirm purchase

    ShoppingAgent->>ShoppingAgent: sign_mandates_on_user_device
    Note over ShoppingAgent: Generate hash of CartMandate and PaymentMandate
    Note over ShoppingAgent: Simulate device signing with user private key

    ShoppingAgent->>ShoppingAgent: Store signed PaymentMandate
    Note over ShoppingAgent: PaymentMandate now has user_authorization

    ShoppingAgent->>CredentialsProvider: A2A Message: Send signed PaymentMandate
    activate CredentialsProvider
    CredentialsProvider->>CredentialsProvider: Store signed PaymentMandate
    CredentialsProvider-->>ShoppingAgent: Acknowledgment
    deactivate CredentialsProvider

    Note over User,PaymentProcessor: Phase 5: Payment Initiation and OTP Challenge

    ShoppingAgent->>MerchantAgent: A2A Message: Initiate payment
    activate MerchantAgent
    Note over MerchantAgent: Validate shopping_agent_id

    MerchantAgent->>PaymentProcessor: A2A Message: Process payment
    activate PaymentProcessor
    PaymentProcessor->>PaymentProcessor: Validate PaymentMandate
    PaymentProcessor->>PaymentProcessor: Check payment method

    PaymentProcessor->>CredentialsProvider: A2A Message: Get payment credentials
    activate CredentialsProvider
    CredentialsProvider->>CredentialsProvider: Verify token and signed mandate
    CredentialsProvider->>CredentialsProvider: Decrypt DPAN card data
    CredentialsProvider-->>PaymentProcessor: Payment credentials
    deactivate CredentialsProvider

    PaymentProcessor->>PaymentProcessor: Initiate OTP challenge
    Note over PaymentProcessor: Generate OTP request

    PaymentProcessor-->>MerchantAgent: Challenge required
    deactivate PaymentProcessor
    MerchantAgent-->>ShoppingAgent: OTP challenge request
    deactivate MerchantAgent

    ShoppingAgent-->>User: Enter verification code
    Note over User: Display OTP challenge text

    Note over User,PaymentProcessor: Phase 6: OTP Verification and Payment Completion

    User->>ShoppingAgent: Provide OTP code
    ShoppingAgent->>MerchantAgent: A2A Message: Initiate payment with OTP
    activate MerchantAgent

    MerchantAgent->>PaymentProcessor: A2A Message: Process payment with challenge response
    activate PaymentProcessor
    PaymentProcessor->>PaymentProcessor: Validate OTP code
    PaymentProcessor->>PaymentProcessor: Process payment transaction
    Note over PaymentProcessor: Charge card via payment network
    PaymentProcessor->>PaymentProcessor: Generate transaction ID
    PaymentProcessor-->>MerchantAgent: Payment SUCCESS
    deactivate PaymentProcessor

    MerchantAgent-->>ShoppingAgent: Payment confirmation
    deactivate MerchantAgent

    ShoppingAgent-->>User: Payment Receipt
    Note over User: Display receipt with transaction details
    deactivate ShoppingAgent
```

## Key Components

### 1. IntentMandate Structure
The IntentMandate captures the user's shopping intent:

```json
{
  "intent_mandate_id": "uuid",
  "natural_language_description": "I want to buy a coffee maker",
  "user_prompt_required": true,
  "merchants": ["merchant_agent"],
  "skus": ["coffee-maker-001"],
  "intent_expiry": "2025-11-18T10:00:00Z",
  "requires_refundability": false
}
```

### 2. CartMandate Structure
The CartMandate contains product and payment request details:

```json
{
  "cart_mandate_id": "uuid",
  "merchant_name": "Example Merchant",
  "cart_expiry": "2025-11-18T10:00:00Z",
  "refund_period": "P30D",
  "payment_request": {
    "method_data": [{
      "supported_methods": ["CARD"]
    }],
    "details": {
      "id": "order-123",
      "total": {
        "label": "Total",
        "amount": {"currency": "USD", "value": "79.99"}
      },
      "displayItems": [
        {"label": "Coffee Maker", "amount": {"value": "69.99"}},
        {"label": "Shipping", "amount": {"value": "5.00"}},
        {"label": "Tax", "amount": {"value": "5.00"}}
      ]
    }
  },
  "merchant_signature": "signature_by_merchant"
}
```

### 3. PaymentMandate Structure
The PaymentMandate authorizes the payment:

```json
{
  "payment_mandate_contents": {
    "payment_mandate_id": "uuid",
    "timestamp": "2025-11-17T12:00:00Z",
    "payment_details_id": "order-123",
    "payment_details_total": {
      "label": "Total",
      "amount": {"currency": "USD", "value": "79.99"}
    },
    "payment_response": {
      "request_id": "order-123",
      "method_name": "CARD",
      "details": {
        "token": {
          "value": "encrypted_token",
          "url": "http://localhost:8002/a2a/credentials_provider"
        }
      },
      "shipping_address": {
        "streetAddress": "123 Main St",
        "city": "San Francisco",
        "state": "CA",
        "zipCode": "94102"
      },
      "payer_email": "user@example.com"
    },
    "merchant_agent": "merchant_agent"
  },
  "user_authorization": "cart_hash_payment_hash_signature"
}
```

### 4. Payment Credential Token
The token contains DPAN card data:

```json
{
  "value": "encrypted_payment_token",
  "url": "http://localhost:8002/a2a/credentials_provider"
}
```

The Credentials Provider decrypts this to reveal:

```json
{
  "card_number": "4111111111111111",
  "expiry_month": "12",
  "expiry_year": "2025",
  "cvv": "123",
  "holder_name": "John Doe",
  "is_dpan": true
}
```

### 5. OTP Challenge
The OTP challenge structure:

```json
{
  "challenge": {
    "type": "otp",
    "display_text": "The payment method issuer sent a verification code to the phone number on file..."
  }
}
```

## Protocol Flow Details

### Mandate Lifecycle

1. **IntentMandate**: Created by Shopping Agent to express user's shopping intent
2. **CartMandate**: Created by Merchant Agent with product details and pricing
3. **PaymentMandate**: Created by Shopping Agent with selected payment method

### Security Features

- **Merchant Signature**: CartMandate is signed by merchant to ensure authenticity
- **User Authorization**: PaymentMandate is signed by user to authorize purchase
- **Hash Binding**: User signature includes hashes of both CartMandate and PaymentMandate
- **Token-Based Credentials**: Payment details are never directly shared, only tokens
- **DPAN**: Tokenized card number instead of primary account number (PAN)
- **OTP Challenge**: Additional authentication layer for high-confidence transactions

### Agent Responsibilities

| Agent | Responsibilities |
|-------|-----------------|
| **Shopping Agent** | Orchestrates flow, manages user interaction, creates IntentMandate and PaymentMandate |
| **Merchant Agent** | Product catalog, creates and signs CartMandate, validates shopping_agent_id |
| **Credentials Provider** | Manages payment methods, provides DPAN tokens, decrypts credentials |
| **Payment Processor** | Processes payments, implements OTP challenge, charges card |

### Protocol Standards

This implementation follows several key standards:

1. **A2A (Agent-to-Agent)**: Communication protocol between agents
2. **AP2 (Agent Payments Protocol)**: Payment-specific extension to A2A
3. **Mandate Pattern**: Structured, signed data objects for cart and payment
4. **DPAN**: Tokenized card numbers for enhanced security
5. **JSON-RPC 2.0**: Message format for agent communication

### Notes

- This is a demonstration implementation showing the card payment flow
- The signing simulation uses simple hash concatenation instead of real cryptographic signatures
- OTP validation uses a hardcoded value (123) for demo purposes
- Production implementations should:
  - Use real cryptographic signatures (e.g., JWT, JWS)
  - Implement proper certificate validation
  - Use secure OTP generation and validation
  - Add fraud detection mechanisms
  - Implement proper error handling and retry logic
  - Store sensitive data securely (encrypted at rest)
  - Use secure communication channels (TLS/HTTPS)


## AI 에이전트 통합 방식

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

