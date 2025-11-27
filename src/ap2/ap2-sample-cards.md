# Cards 시나리오

```{warning}
**문서 작성 중**

이 문서와 하위 문서들은 현재 작성 중이며, 부정확하거나 불완전한 정보를 포함할 수 있습니다. 
프로덕션 환경에서 사용하기 전에 반드시 공식 문서와 표준을 참조하시기 바랍니다.
```

Human-Present flows refer to all commerce flows where the user is present to confirm the details of what is being purchased, and what payment method is to be used. The user attesting to the details of the purchase allows all parties to have high confidence of the transaction.

The IntentMandate is still leveraged to share the appropriate information with Merchant Agents. This is to maintain consistency across Human-Present and Human-Not-Present flows.

All Human-Present purchases will have a user-signed PaymentMandate authorizing the purchase.

## Key Actors

This sample consists of:

*   **Shopping Agent:** The main orchestrator that handles user's requests to
    shop and delegates tasks to specialized agents.
*   **Merchant Agent:** An agent that handles product queries from the shopping
    agent.
*   **Merchant Payment Processor Agent:** An agent that takes payments on behalf
    of the merchant.
*   **Credentials Provider Agent:** The credentials provider is the holder of a
    user's payment credentials. As such, it serves two primary roles:
    *   It provides the shopping agent the list of payment methods available in
        a user's wallet.
    *   It facilitates payment between the shopping agent and a merchant's
        payment processor.

## Key Features

**1. Card purchase with DPAN**

*   The merchant agent will advertise support for card purchases through its
    agent card and through the CartMandate once shopping is complete.
*   The preferred payment method in the user's wallet will be a tokenized (DPAN)
    card.

**2. OTP Challenge**

*   The merchant payment processor agent will request an OTP challenge of the
    user in order to complete payment.

## Sequence Diagram

```{mermaid}
sequenceDiagram
    participant User
    participant ShoppingAgent as Shopping Agent
    participant MerchantAgent as Merchant Agent
    participant CredentialsProvider as Credentials Provider
    participant PaymentProcessor as Payment Processor

    Note over User,PaymentProcessor: Phase 1: Initial Request and Cart Creation

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

    Note over User,PaymentProcessor: Phase 2: Product Selection

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

### Interacting with the Shopping Agent

1.  **Initial Request**: In the Shopping Agent's terminal, you'll be prompted to
    start a conversation. You can type something like: "I want to buy a coffee
    maker."
1.  **Product Search**: The Shopping Agent will delegate to the Merchant Agent,
    which will find products matching your intent and present you with options
    contained in CartMandates.
1.  **Cart Creation**: The Merchant Agent will create one or more `CartMandate`s
    and share it with the Shopping Agent. Each CartMandate is signed by the
    Merchant, ensuring the offer to the user is accurate.
1.  **Product Selection** The Shopping Agent will present the user with the set
    of products to choose from.
1.  **Link Credential Provider**: The Shopping Agent will prompt you to link
    your preferred Credential Provider in order to access you available payment
    methods.
1.  **Payment Method Selection**: After you select a cart, the Shopping Agent
    will show you a list of available payment methods from the Credentials
    Provider Agent. You will select a payment method.
1.  **PaymentMandate creation**: The Shopping Agent will package the cart and
    transaction information in a PaymentMandate and ask you to sign the
    mandate. It will initiate payment using the PaymentMandate.
1.  **OTP Challenge**: The Merchant Payment Processor will then request an OTP,
    and you'll be asked to provide a mock OTP to the agent. Use `123`
1.  **Purchase Complete**: Once the OTP is provided, the payment will be
    processed, and you'll receive a confirmation message and a digital receipt.

## Key Components

### 1. IntentMandate Structure

The IntentMandate captures the user's shopping intent:

```json
{
  "user_cart_confirmation_required": true,
  "natural_language_description": "I want to buy a coffee maker",
  "merchants": ["merchant_agent"],
  "skus": ["coffee-maker-001"],
  "requires_refundability": false,
  "intent_expiry": "2025-11-18T10:00:00Z"
}
```

**Key Fields:**
- `user_cart_confirmation_required`: If false, the agent can make purchases automatically once conditions are satisfied
- `natural_language_description`: User's intent in natural language, confirmed by the user
- `merchants`: Optional list of allowed merchants (null = any merchant)
- `skus`: Optional list of specific product SKUs (null = any SKU)
- `requires_refundability`: Whether items must be refundable
- `intent_expiry`: Expiration time in ISO 8601 format

**Note:** In actual A2A messages, the IntentMandate is wrapped in a Message structure that includes additional metadata:
- `contextId` (shopping context identifier)
- `messageId`, `taskId`, `role` (A2A protocol fields)
- `risk_data` (optional, sent as a separate DataPart for fraud prevention)

### 2. CartMandate Structure
The CartMandate is a merchant-signed cart whose contents are guaranteed:

```json
{
  "contents": {
    "id": "cart_123",
    "user_cart_confirmation_required": true,
    "merchant_name": "Example Merchant",
    "cart_expiry": "2025-11-18T10:00:00Z",
    "payment_request": {
      "method_data": [{
        "supported_methods": "CARD",
        "data": {}
      }],
      "details": {
        "id": "order-123",
        "total": {
          "label": "Total",
          "amount": {"currency": "USD", "value": "79.99"}
        },
        "display_items": [
          {
            "label": "Coffee Maker",
            "amount": {"currency": "USD", "value": "69.99"},
            "refund_period": 30
          },
          {"label": "Shipping", "amount": {"currency": "USD", "value": "5.00"}},
          {"label": "Tax", "amount": {"currency": "USD", "value": "5.00"}}
        ]
      }
    }
  },
  "merchant_authorization": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjIwMjQwOTA..."
}
```

**Key Fields:**
- `contents`: The CartContents object containing cart details
  - `id`: Unique cart identifier
  - `user_cart_confirmation_required`: Whether user must confirm before purchase
  - `merchant_name`: Name of the merchant
  - `cart_expiry`: Expiration time in ISO 8601 format
  - `payment_request`: W3C PaymentRequest object with items, prices, and accepted payment methods
- `merchant_authorization`: Optional JWT (base64url-encoded) signing the cart contents
  - Header: signing algorithm and key ID
  - Payload: `iss`, `sub`, `aud`, `iat`, `exp`, `jti`, `cart_hash`
  - Signature: Merchant's digital signature for authenticity verification

### 3. PaymentMandate Structure
The PaymentMandate contains user's payment authorization and provides transaction visibility to the payment ecosystem:

```json
{
  "payment_mandate_contents": {
    "payment_mandate_id": "pm_uuid_123",
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
        "street_address": "123 Main St",
        "city": "San Francisco",
        "state": "CA",
        "zip_code": "94102"
      },
      "payer_email": "user@example.com"
    },
    "merchant_agent": "merchant_agent",
    "timestamp": "2025-11-17T12:00:00Z"
  },
  "user_authorization": "eyJhbGciOiJFUzI1NksiLCJraWQiOiJkaWQ6ZXhhbXBsZ..."
}
```

**Key Fields:**
- `payment_mandate_contents`: Core payment data
  - `payment_mandate_id`: Unique payment mandate identifier
  - `payment_details_id`: Payment request identifier
  - `payment_details_total`: Total payment amount (PaymentItem)
  - `payment_response`: PaymentResponse with selected payment method details
  - `merchant_agent`: Merchant identifier
  - `timestamp`: Mandate creation time (ISO 8601)
- `user_authorization`: Optional base64url-encoded verifiable presentation (e.g., sd-jwt-vc)
  - Issuer-signed JWT authorizing a 'cnf' claim
  - Key-binding JWT with `aud`, `nonce`, `sd_hash`
  - `transaction_data`: Array containing secure hashes of CartMandate and PaymentMandateContents
  - Purpose: Helps network/issuer build trust in agentic transactions

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

