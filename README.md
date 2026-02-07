# KalqiX API -- Quick Start Guide

Welcome to the integration guide. This document will help you authenticate, access market data, manage orders, and monitor your account using our API.

---

### Web App

Get your api key and secret from kalqix app [user > settings > api keys].

```
https://testnet.kalqix.com
```

### Primary API Endpoint
```
https://testnet-api.kalqix.com/v1/
```

### Required Headers
```javascript
const headers = {
    'x-api-key': 'your-api-key',
    'x-api-signature': 'hmac-signature',
    'x-api-timestamp': 'timestamp-in-ms',
    'Content-Type': 'application/json'
};
```

### HMAC Signature Generation
We use HMAC so the server can verify that a request was created by someone who holds your API secret without ever sending the secret over the network.
```javascript
const crypto = require('crypto');

function signRequest(method, path, body, timestamp, apiSecret) {
    let payload = '';
    if (body && Object.keys(body).length > 0) {
        payload = canonicalizePayload(body);
    }
    const canonical = `${method}|${path}|${payload}|${timestamp}`;
    return crypto.createHmac('sha256', apiSecret).update(canonical).digest('hex');
}

// Example usage
const method = 'POST';
const path = '/orders';
const body = { ticker: 'BTC_USDT', price: '100000', ... };
const timestamp = Date.now();
const signature = signRequest(method, path, body, timestamp, apiSecret);
```

### Ethereum Wallet Signature (Message Signing)

Certain endpoints require an Ethereum wallet signature in addition to HMAC authentication. The signed payload is a canonicalized JSON string with sorted keys that includes an `action` field.

**Canonicalization:** `JSON.stringify(payload, Object.keys(payload).sort())`

```javascript
const { ethers } = require('ethers');

function canonicalizePayload(payload) {
    return JSON.stringify(payload, Object.keys(payload).sort());
}

async function signPayload(payload, walletSeed) {
    const wallet = ethers.Wallet.fromPhrase(walletSeed);
    const signingString = canonicalizePayload(payload);
    const signature = await wallet.signMessage(signingString);
    return { signature, walletAddress: wallet.address };
}
```

**Timestamp rules:**
- Must be an integer in milliseconds (not ISO string)
- Rejected if in the future or older than 5 minutes
- Replay protection is enforced (same signed message cannot be reused)

### Complete Order Placement Example
```javascript
const crypto = require('crypto');
const { ethers } = require('ethers');

const API_KEY = process.env.API_KEY;
const API_SECRET = process.env.API_SECRET;
const WALLET_SEED = process.env.WALLET_SEED;

function canonicalizePayload(payload) {
    return JSON.stringify(payload, Object.keys(payload).sort());
}

function signRequest(method, path, body, timestamp, apiSecret) {
    let payload = '';
    if (body && Object.keys(body).length > 0) {
        payload = canonicalizePayload(body);
    }
    const canonical = `${method}|${path}|${payload}|${timestamp}`;
    return crypto.createHmac('sha256', apiSecret).update(canonical).digest('hex');
}

async function placeOrder(orderData) {
    const wallet = ethers.Wallet.fromPhrase(WALLET_SEED);
    const timestamp = Date.now();

    // 1. Build the payload
    const payload = {
        ticker: orderData.ticker,
        side: orderData.side,
        order_type: orderData.order_type,
        quantity: orderData.quantity,
        quote_quantity: orderData.quote_quantity || '',
        price: orderData.price || '',
        time_in_force: orderData.time_in_force || 0,
        expires_at: orderData.expires_at || 0,
        timestamp
    };

    // 2. Sign with wallet (same payload + action)
    const signingString = canonicalizePayload({ action: 'PLACE_ORDER', ...payload });
    const walletSignature = await wallet.signMessage(signingString);

    // 3. Request body = payload + signature
    const requestBody = {
        ...payload,
        signature: walletSignature
    };

    // 4. Generate HMAC signature (for headers)
    const method = 'POST';
    const path = '/orders';
    const hmacSignature = signRequest(method, path, requestBody, timestamp, API_SECRET);

    // 4. Send request
    const response = await fetch('https://testnet-api.kalqix.com/v1/orders', {
        method: 'POST',
        headers: {
            'x-api-key': API_KEY,
            'x-api-signature': hmacSignature,
            'x-api-timestamp': timestamp.toString(),
            'Content-Type': 'application/json'
        },
        body: JSON.stringify(requestBody)
    });

    return await response.json();
}

// Example
const result = await placeOrder({
    ticker: 'BTC_USDT',
    price: '100000',
    quantity: '0.1',
    side: 'BUY',
    order_type: 'LIMIT'
});
```

### Alternative Wallet Creation Methods

```javascript
// From private key
const privateKey = '0x...'; // Your private key
const wallet = new ethers.Wallet(privateKey);

// From mnemonic (same as seed phrase)
const mnemonic = 'your twelve word mnemonic here';
const wallet = ethers.Wallet.fromPhrase(mnemonic);

// Generate a new random wallet
const randomWallet = ethers.Wallet.createRandom();
```

---

## 1. Authentication

### API Key Authentication

For programmatic access, use API key authentication with HMAC signatures.

#### Getting Your API Keys:
- **Dashboard:** Visit Kalqix Settings > API Keys dashboard to create your first API keys
- **Self-Service:** Once you have API keys, you can use the `PUT /api-keys` endpoint to create additional keys

---

## 2. Market Data Endpoints

- `GET /markets` -- List all markets
- `GET /markets/:ticker` -- Market details
- `GET /markets/:ticker/order-book` -- Orderbook snapshot
- `GET /markets/:ticker/price` -- Current price
- `GET /markets/:ticker/market-price` -- Market order price
- `GET /markets/:ticker/orders` -- All orders for a market
- `GET /markets/:ticker/trades` -- Recent trades
- `GET /markets/:ticker/line-chart` -- Line chart data
- `GET /markets/:ticker/candles` -- OHLCV candles
- `GET /markets/:ticker/24hr` -- 24hr stats
- `GET /markets/:ticker/volume` -- Market volume

**Example:**
```http
GET /markets/BTC_USDT/order-book
```

---

## 3. Order Management

- `POST /orders` -- Place a new order
- `GET /orders` -- List your orders
- `GET /orders/:id` -- Order status/details
- `DELETE /orders/:id` -- Cancel an order (query params: `order_id`, `timestamp`, `signature`)
- `DELETE /users/me/cancel-all-orders` -- Cancel all orders (query params: `market`, `timestamp`, `signature`)

**For comprehensive order management documentation, see [ORDER_MANAGEMENT_API.md](./ORDER_MANAGEMENT_API.md)**

---

## 4. Withdrawals

- `GET /withdrawals` -- List your withdrawals
- `GET /withdrawals/:id` -- Withdrawal details
- `GET /withdrawals/:id/claim` -- Get claim data for a withdrawal
- `POST /withdrawals` -- Create a withdrawal

### Create Withdrawal

```
POST /withdrawals
```

**Request Body:**
```javascript
{
    "asset": "USDC",              // Required: Asset symbol
    "amount": "100",              // Required: Formatted amount (e.g. "100" USDC)
    "chain_id": 84532,            // Required: Destination chain ID
    "timestamp": 1700000000000,   // Required: Integer milliseconds
    "signature": "0x..."          // Required: Wallet signature
}
```

**Wallet signing payload:**
```javascript
const payload = {
    action: 'WITHDRAW',
    asset: 'USDC',
    amount: '100',
    chain_id: 84532,
    timestamp: 1700000000000
};
const signingString = canonicalizePayload(payload);
const signature = await wallet.signMessage(signingString);
```

---

## 5. Transfers

- `GET /transfers` -- List your transfers
- `GET /transfers/:id` -- Transfer details
- `PUT /transfers` -- Create a transfer

### Create Transfer

```
PUT /transfers
```

**Request Body:**
```javascript
{
    "asset": "USDC",                              // Required: Asset symbol
    "amount": "50",                               // Required: Formatted amount
    "to_wallet_address": "0x1234...abcd",         // Required: Destination wallet address
    "timestamp": 1700000000000,                   // Required: Integer milliseconds
    "signature": "0x..."                          // Required: Wallet signature
}
```

**Wallet signing payload:**
```javascript
const payload = {
    action: 'TRANSFER',
    asset: 'USDC',
    amount: '50',
    to_wallet_address: '0x1234...abcd',
    timestamp: 1700000000000
};
const signingString = canonicalizePayload(payload);
const signature = await wallet.signMessage(signingString);
```

---

## 6. Account & Portfolio

- `GET /portfolios` -- Get balances
- `GET /positions` -- All positions
- `GET /positions/:asset` -- Position for a specific asset

---

## 7. User & Transaction Info

- `GET /users/me` -- User profile
- `GET /users/:user/transaction-history` -- Transaction history
- `GET /users/:user/trades` -- Trade history

---

## 8. API Key Management

- `GET /api-keys` -- List your API keys
- `PUT /api-keys` -- Create a new API key
- `DELETE /api-keys/:api_key` -- Delete an API key

---

## 9. Rate Limiting

- **Default:** 1000 requests/sec per IP
- **Headers:**
  - `X-RateLimit-Limit`
  - `X-RateLimit-Remaining`
  - `X-RateLimit-Reset`
  - `Retry-After` (on 429)
- **On exceeding:** HTTP 429 with `Retry-After`

---

## 10. Error Handling

- Standard HTTP status codes
- Error responses in JSON: `{ "error": "description" }`

---

## 11. Signing Payloads Reference

All signed-message endpoints use the same pattern: canonicalized JSON with sorted keys.

| Endpoint | Action | Payload Fields |
|----------|--------|----------------|
| `POST /orders` | `PLACE_ORDER` | `ticker`, `side`, `order_type`, `quantity`, `quote_quantity`, `price`, `time_in_force`, `expires_at`, `timestamp` |
| `DELETE /orders/:id` | `CANCEL_ORDER` | `order_id`, `timestamp` |
| `DELETE /users/me/cancel-all-orders` | `CANCEL_ALL_ORDERS` | `market` (integer), `timestamp` |
| `POST /withdrawals` | `WITHDRAW` | `asset`, `amount`, `chain_id`, `timestamp` |
| `PUT /transfers` | `TRANSFER` | `asset`, `amount`, `to_wallet_address`, `timestamp` |

---

## 12. Security Notes

- **Never send your API secret in requests** - only use it to generate signatures
- **Keep your API secret secure** - it's only shown once when created
- **Use HTTPS** for all API requests
- **Timestamp validation** - requests are rejected if timestamp is more than 5 minutes old
- **Rotate API keys regularly** for enhanced security
- **Wallet Seed Security** - Store your wallet seed phrase securely and never expose it in client-side code or logs
- **Hardware Wallets** - Consider using hardware wallets for enhanced security.

---

## 13. Support

For integration help, contact us on x (twitter) @kalqix

---

## 14. Integration Checklist

- [ ] Get initial API keys via dashboard
- [ ] Implement HMAC signature generation
- [ ] Implement wallet message signing (canonicalized JSON with sorted keys)
- [ ] Fetch market/orderbook/trade data
- [ ] Place/cancel orders and check order status
- [ ] Create withdrawals and transfers
- [ ] Fetch balances and positions
- [ ] Handle rate limits and errors gracefully
- [ ] Test with different market pairs

---

*For further technical details, please contact support or see the next steps in this onboarding series.*
