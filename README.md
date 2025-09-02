# KalqiX API -- Quick Start Guide

Welcome to the integration guide. This document will help you authenticate, access market data, manage orders, and monitor your account using our API.

---

### Web App

Get your api key and secret from kalqix app [user > settings > api keys].

```
https://dev.kalqix.com
```

### Primary API Endpoint
```
https://devapi.kalqix.com/v1/
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
    const canonical = `${method}|${path}|${JSON.stringify(body)}|${timestamp}`;
    return crypto.createHmac('sha256', apiSecret).update(canonical).digest('hex');
}

// Example usage
const method = 'POST';
const path = '/orders';
const body = { ticker: 'BTC_USDT', price: '100000', ... };
const timestamp = Date.now();
const signature = signRequest(method, path, body, timestamp, apiSecret);
```

### Ethereum Message Signature for Orders
For order placement, you also need to sign a message with your wallet. This signature goes in the request body, not headers
```javascript
const { ethers } = require('ethers');

async function signOrderMessage(order, walletSeed) {
    // Create wallet from seed phrase
    const wallet = ethers.Wallet.fromPhrase(walletSeed);
    
    // Optional: Connect to a provider if you need to interact with the blockchain
    // const provider = new ethers.JsonRpcProvider('YOUR_RPC_URL');
    // const connectedWallet = wallet.connect(provider);

    const price = order.order_type === 'LIMIT' 
        ? order.price 
        : 'MARKET';    
    const message = `${order.side} ${order.quantity} ${order.ticker} @PRICE: ${price}`;
    const signature = await wallet.signMessage(message);
    const address = wallet.address;
    
    return { message, signature, walletAddress: address };
}

// Example usage
const order = {
    ticker: 'BTC_USDT',
    price: '100000',
    quantity: '0.1',
    side: 'BUY',
    order_type: 'LIMIT'
};

// Your wallet seed phrase (12, 15, 18, 21, or 24 words)
const walletSeed = 'your twelve word seed phrase here for wallet creation';

const { message, signature, walletAddress } = await signOrderMessage(order, walletSeed);
```

### Alternative Wallet Creation Methods

You can also create wallets using other methods:

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

### Complete Order Placement Example
```javascript
// 1. Generate Ethereum message signature (for request body)
const { message, signature, walletAddress } = await signOrderMessage(order, walletSeed);

// 2. Prepare request body with Ethereum signature
const requestBody = {
    ticker: 'BTC_USDT',
    price: '100000',
    quantity: '0.1',
    side: 'BUY',
    order_type: 'LIMIT',
    message: message,
    signature: signature
};

// 3. Generate HMAC signature (for headers)
const method = 'POST';
const path = '/orders';
const timestamp = Date.now();
const hmacSignature = signRequest(method, path, requestBody, timestamp, apiSecret);

// 4. Send request with both signatures
fetch('https://devapi.kalqix.com/v1/orders', {
    method: 'POST',
    headers: {
        'x-api-key': apiKey,
        'x-api-signature': hmacSignature,        // HMAC for API auth
        'x-api-timestamp': timestamp.toString(),
        'Content-Type': 'application/json'
    },
    body: JSON.stringify(requestBody)            // Contains Ethereum signature
});
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

- `GET /markets` — List all markets
- `GET /markets/:ticker` — Market details
- `GET /markets/:ticker/order-book` — Orderbook snapshot
- `GET /markets/:ticker/price` — Current price
- `GET /markets/:ticker/market-price` — Market order price
- `GET /markets/:ticker/orders` — All orders for a market
- `GET /markets/:ticker/trades` — Recent trades
- `GET /markets/:ticker/line-chart` — Line chart data
- `GET /markets/:ticker/candles` — OHLCV candles
- `GET /markets/:ticker/24hr` — 24hr stats
- `GET /markets/:ticker/volume` — Market volume

**Example:**
```http
GET /markets/BTC_USDT/order-book
```

---

## 3. Order Management

- `POST /orders` — Place a new order
- `GET /orders` — List your orders
- `GET /orders/:id` — Order status/details
- `DELETE /orders/:id` — Cancel an order
- `DELETE /orders/:id/all` — Cancel all orders

**📖 For comprehensive order management documentation, see [ORDER_MANAGEMENT_API.md](./ORDER_MANAGEMENT_API.md)**

---

## 4. Account & Portfolio

- `GET /portfolios` — Get balances
- `GET /positions` — All positions
- `GET /positions/:asset` — Position for a specific asset
- `POST /positions/:asset/withdraw` — Withdraw from a position

---

## 5. User & Transaction Info

- `GET /users/me` — User profile
- `GET /users/:user/transaction-history` — Transaction history
- `GET /users/:user/trades` — Trade history

---

## 6. Rate Limiting

- **Default:** 200 requests/sec per IP
- **Headers:**
  - `X-RateLimit-Limit`
  - `X-RateLimit-Remaining`
  - `X-RateLimit-Reset`
  - `Retry-After` (on 429)
- **On exceeding:** HTTP 429 with `Retry-After`

---

## 7. Error Handling

- Standard HTTP status codes
- Error responses in JSON: `{ "error": "description" }`

---

## 8. Security Notes

- **Never send your API secret in requests** - only use it to generate signatures
- **Keep your API secret secure** - it's only shown once when created
- **Use HTTPS** for all API requests
- **Timestamp validation** - requests are rejected if timestamp is more than 5 minutes old
- **Rotate API keys regularly** for enhanced security
- **Wallet Seed Security** - Store your wallet seed phrase securely and never expose it in client-side code or logs
- **Hardware Wallets** - Consider using hardware wallets for enhanced security.

---

## 9. Support

For integration help, contact us on x (twitter) @kalqix

---

## 10. Integration Checklist

- [ ] Get initial API keys via dashboard
- [ ] Implement HMAC signature generation
- [ ] Fetch market/orderbook/trade data
- [ ] Place/cancel orders and check order status
- [ ] Fetch balances and positions
- [ ] Handle rate limits and errors gracefully
- [ ] Test with different market pairs

---

*For further technical details, please contact support or see the next steps in this onboarding series.* 
