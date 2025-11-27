# x402 시나리오

```{warning}
**문서 작성 중**

이 문서와 하위 문서들은 현재 작성 중이며, 부정확하거나 불완전한 정보를 포함할 수 있습니다. 
프로덕션 환경에서 사용하기 전에 반드시 공식 문서와 표준을 참조하시기 바랍니다.
```

This illustrates the complete flow of the Human Present x402 Payment scenario using the AP2 framework with the A2A protocol and x402 payment standard.

## Sequence Diagram

```{mermaid}
sequenceDiagram
    participant User
    participant ShoppingAgent
    participant MerchantAgent
    participant CredentialsProvider
    participant PaymentProcessor
    participant Blockchain

    Note right of User: Phase 1: Product Discovery

    User->>ShoppingAgent: I want to buy a coffee maker
    ShoppingAgent->>ShoppingAgent: Create IntentMandate
    ShoppingAgent->>MerchantAgent: A2A Message: Find products
    MerchantAgent->>MerchantAgent: Validate shopping_agent_id
    MerchantAgent->>MerchantAgent: Check x402 support
    MerchantAgent->>MerchantAgent: Search product catalog
    MerchantAgent->>MerchantAgent: Create CartMandate with x402
    MerchantAgent-->>ShoppingAgent: CartMandate with products
    ShoppingAgent-->>User: Display product options
    User->>ShoppingAgent: Select product

    Note right of User: Phase 2: Shipping Address

    User->>ShoppingAgent: Provide shipping address
    ShoppingAgent->>MerchantAgent: A2A Message: Update cart
    MerchantAgent->>MerchantAgent: Update and re-sign CartMandate
    MerchantAgent-->>ShoppingAgent: Updated CartMandate

    Note right of User: Phase 3: x402 Payment Method

    ShoppingAgent->>CredentialsProvider: Get payment methods
    CredentialsProvider->>CredentialsProvider: Filter x402 methods
    CredentialsProvider->>CredentialsProvider: Check wallet balances
    CredentialsProvider-->>ShoppingAgent: x402 payment methods
    ShoppingAgent-->>User: Display digital currencies
    User->>ShoppingAgent: Select digital currency
    ShoppingAgent->>CredentialsProvider: Get payment token
    CredentialsProvider->>CredentialsProvider: Generate x402 token
    CredentialsProvider-->>ShoppingAgent: x402 credential token

    Note right of User: Phase 4: Payment Mandate

    ShoppingAgent->>ShoppingAgent: Create PaymentMandate
    ShoppingAgent-->>User: Display final details with gas fee
    User->>ShoppingAgent: Confirm purchase
    ShoppingAgent->>ShoppingAgent: sign_mandates_on_user_device
    ShoppingAgent->>ShoppingAgent: Generate mandate hashes
    ShoppingAgent->>CredentialsProvider: Send signed PaymentMandate
    CredentialsProvider->>CredentialsProvider: Validate wallet balance
    CredentialsProvider-->>ShoppingAgent: Acknowledgment

    Note right of User: Phase 5: Payment Initiation

    ShoppingAgent->>MerchantAgent: A2A Message: Initiate payment
    MerchantAgent->>PaymentProcessor: Process x402 payment
    PaymentProcessor->>PaymentProcessor: Validate PaymentMandate
    PaymentProcessor->>CredentialsProvider: Get payment credentials
    CredentialsProvider->>CredentialsProvider: Retrieve wallet credentials
    CredentialsProvider-->>PaymentProcessor: Wallet address and signing key
    PaymentProcessor->>PaymentProcessor: Prepare blockchain transaction
    PaymentProcessor->>PaymentProcessor: Sign transaction

    Note right of User: Phase 6: Blockchain Execution

    PaymentProcessor->>Blockchain: Submit transaction
    Blockchain->>Blockchain: Validate transaction
    Blockchain->>Blockchain: Execute smart contract
    Blockchain->>Blockchain: Deduct gas fee
    Blockchain->>Blockchain: Transfer digital currency
    Blockchain->>Blockchain: Confirm in block
    Blockchain-->>PaymentProcessor: Transaction hash and receipt
    PaymentProcessor->>PaymentProcessor: Verify confirmation
    PaymentProcessor->>PaymentProcessor: Generate transaction ID
    PaymentProcessor-->>MerchantAgent: Payment SUCCESS
    MerchantAgent-->>ShoppingAgent: Confirmation with tx hash
    ShoppingAgent-->>User: Payment Receipt with blockchain details
```

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

### 2. CartMandate Structure with x402 Support
The CartMandate is a merchant-signed cart for x402 blockchain payments:

```json
{
  "contents": {
    "id": "cart_x402_123",
    "user_cart_confirmation_required": true,
    "merchant_name": "Example Merchant",
    "cart_expiry": "2025-11-18T10:00:00Z",
    "payment_request": {
      "method_data": [{
        "supported_methods": "x402",
        "data": {
          "supported_chains": ["ethereum", "polygon", "base"],
          "supported_currencies": ["USDC", "USDT", "DAI"],
          "merchant_wallet_address": "0x1234567890abcdef..."
        }
      }],
      "details": {
        "id": "order-123",
        "total": {
          "label": "Total",
          "amount": {"currency": "USDC", "value": "79.99"}
        },
        "display_items": [
          {
            "label": "Coffee Maker",
            "amount": {"currency": "USDC", "value": "69.99"},
            "refund_period": 30
          },
          {"label": "Shipping", "amount": {"currency": "USDC", "value": "5.00"}},
          {"label": "Tax", "amount": {"currency": "USDC", "value": "5.00"}},
          {"label": "Estimated Gas Fee", "amount": {"currency": "USDC", "value": "0.50"}}
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
  - `payment_request`: W3C PaymentRequest with x402-specific data
    - `supported_chains`: Blockchain networks accepted
    - `supported_currencies`: Cryptocurrencies accepted
    - `merchant_wallet_address`: Merchant's blockchain wallet address
- `merchant_authorization`: Optional JWT (base64url-encoded) signing the cart contents

### 3. PaymentMandate Structure for x402
The PaymentMandate contains user's x402 payment authorization:

```json
{
  "payment_mandate_contents": {
    "payment_mandate_id": "pm_x402_123",
    "payment_details_id": "order-123",
    "payment_details_total": {
      "label": "Total",
      "amount": {"currency": "USDC", "value": "79.99"}
    },
    "payment_response": {
      "request_id": "order-123",
      "method_name": "x402",
      "details": {
        "token": {
          "value": "encrypted_x402_token",
          "url": "http://localhost:8002/a2a/credentials_provider"
        },
        "chain_id": "1",
        "currency": "USDC",
        "contract_address": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
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
  - `payment_response`: PaymentResponse with x402-specific details
    - `chain_id`: Blockchain network identifier
    - `currency`: Cryptocurrency used (e.g., USDC, USDT, DAI)
    - `contract_address`: Token contract address on the blockchain
  - `merchant_agent`: Merchant identifier
  - `timestamp`: Mandate creation time (ISO 8601)
- `user_authorization`: Optional base64url-encoded verifiable presentation
  - Contains secure hashes of CartMandate and PaymentMandateContents
  - Used by blockchain payment processor to verify transaction authenticity

### 4. x402 Payment Credential Token
The token contains blockchain wallet credentials:

```json
{
  "value": "encrypted_x402_payment_token",
  "url": "http://localhost:8002/a2a/credentials_provider"
}
```

The Credentials Provider decrypts this to reveal:

```json
{
  "wallet_address": "0xabcdef1234567890...",
  "chain_id": "1",
  "currency": "USDC",
  "balance": "1000.00",
  "signing_authority": "delegated_signer_key",
  "is_smart_wallet": true
}
```

### 5. Blockchain Transaction Structure
The transaction submitted to the blockchain:

```json
{
  "from": "0xabcdef1234567890...",
  "to": "0x1234567890abcdef...",
  "value": "0",
  "data": "0xa9059cbb...",
  "chainId": 1,
  "nonce": 42,
  "gasLimit": "100000",
  "maxFeePerGas": "50000000000",
  "maxPriorityFeePerGas": "2000000000"
}
```

### 6. Transaction Receipt
The blockchain receipt confirming the transaction:

```json
{
  "transactionHash": "0x123abc...",
  "blockNumber": 12345678,
  "blockHash": "0x789def...",
  "from": "0xabcdef1234567890...",
  "to": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "gasUsed": "65000",
  "effectiveGasPrice": "50000000000",
  "status": "success"
}
```

## 기술 상세

### HTTP 402 상태 코드

```http
HTTP/1.1 402 Payment Required
Content-Type: application/json
WWW-Authenticate: Bearer realm="payment-api"

{
  "amount": "0.01",
  "currency": "USD",
  "payment_methods": ["crypto", "micropayment", "card"],
  "payment_endpoint": "https://payment.example.com/pay",
  "resource_id": "api-call-12345"
}
```

### 에이전트 자동 결제 로직

```python
async def fetch_with_payment(url: str, agent_wallet):
    response = await http_client.get(url)
    
    if response.status_code == 402:
        payment_info = response.json()
        
        # 자동 결제 판단 로직
        if agent_wallet.can_afford(payment_info['amount']):
            # 결제 실행
            payment_token = await agent_wallet.pay(
                amount=payment_info['amount'],
                endpoint=payment_info['payment_endpoint']
            )
            
            # 결제 증명과 함께 재요청
            response = await http_client.get(
                url,
                headers={'Authorization': f'Bearer {payment_token}'}
            )
    
    return response
```

### 결제 토큰 검증

```python
def verify_payment_token(token: str, resource_id: str):
    try:
        decoded = jwt.decode(token, public_key, algorithms=['RS256'])
        
        # 토큰 유효성 확인
        assert decoded['resource_id'] == resource_id
        assert decoded['exp'] > time.time()
        assert decoded['amount_paid'] >= required_amount
        
        return True
    except:
        return False
```

