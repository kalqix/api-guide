# Order Management API Documentation

This document provides comprehensive guidance for managing orders through the Kalqix API, including placement, cancellation, and status retrieval.

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

## Table of Contents

1. [Authentication](#authentication)
2. [Order Placement](#order-placement)
3. [Order Status Retrieval](#order-status-retrieval)
4. [Order Cancellation](#order-cancellation)
5. [Cancel All Orders](#cancel-all-orders)
6. [Order Types and Statuses](#order-types-and-status)
7. [Error Handling](#error-handling)
8. [Rate Limiting](#rate-limiting)

---

## Authentication

All order management endpoints require API key authentication with HMAC signatures.

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
```

### Wallet Message Signing

Order placement and cancellation require Ethereum wallet signatures. The signed payload is a canonicalized JSON string with sorted keys.

```javascript
function canonicalizePayload(payload) {
    return JSON.stringify(payload, Object.keys(payload).sort());
}
```

**Timestamp rules:**
- Must be an integer in milliseconds (not ISO string)
- Rejected if in the future or older than 5 minutes
- Replay protection is enforced

---

## Order Placement

### Endpoint
```
POST /orders
```

### Request Body
```javascript
{
    "ticker": "BTC_USDT",           // Required: Market pair
    "price": "30000",               // Required for LIMIT orders
    "quantity": "0.1",              // Required: Order quantity
    "side": "BUY",                  // Required: BUY or SELL
    "order_type": "LIMIT",          // Required: LIMIT or MARKET
    "time_in_force": 0,             // Optional: 0=GTC, 1=IOC, 2=FOK
    "expires_at": 0,                // Optional: Expiry timestamp in ms
    "quote_quantity": "",           // Optional: For market buy orders with budget
    "signature": "0x...",           // Required: Wallet signature
    "timestamp": 1700000000000      // Required: Integer milliseconds
}
```

### Wallet Signing Payload

```javascript
const payload = {
    action: 'PLACE_ORDER',
    ticker: 'BTC_USDT',
    side: 'BUY',
    order_type: 'LIMIT',
    quantity: '0.1',
    quote_quantity: '',
    price: '30000',
    time_in_force: 0,
    expires_at: 0,
    timestamp: 1700000000000
};
const signingString = canonicalizePayload(payload);
const signature = await wallet.signMessage(signingString);
```

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

    // 4. Generate HMAC signature
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
    price: '30000',
    quantity: '0.1',
    side: 'BUY',
    order_type: 'LIMIT'
});
console.log('Order placed:', result);
```

### Response Format

#### Success Response (201)
```javascript
{
    "order_id": "ord_1234567890"
}
```

#### Error Responses

**400 Bad Request**
```javascript
{
    "error": "Missing required fields."
}
```

**401 Unauthorized**
```javascript
{
    "error": "Invalid API key or signature."
}
```

**403 Forbidden**
```javascript
{
    "error": "Invalid message signature."
}
```

**404 Not Found**
```javascript
{
    "error": "Invalid market."
}
```

---

## Order Status Retrieval

### Get Single Order

#### Endpoint
```
GET /orders/:orderId
```

#### Example Request
```javascript
const method = 'GET';
const path = '/orders/ord_1234567890';
const body = {};
const timestamp = Date.now();
const signature = signRequest(method, path, body, timestamp, apiSecret);

const response = await fetch('https://testnet-api.kalqix.com/v1/orders/ord_1234567890', {
    method: 'GET',
    headers: {
        'x-api-key': apiKey,
        'x-api-signature': signature,
        'x-api-timestamp': timestamp.toString(),
        'Content-Type': 'application/json'
    }
});
```

#### Response Format
```javascript
{
    "order_id": "ord_1234567890",
    "ticker": "BTC/USDT",
    "price": "30000",
    "quantity": "0.1",
    "remaining_quantity": "0.05",
    "side": "BUY",
    "order_type": "LIMIT",
    "status": "PARTIALLY_FILLED",
    "timestamp": "2024-01-15T10:30:00Z",
    "wallet_address": "0x1234...",
    "token": "BTC",
    "quote_asset": "USDT",
    "trades": [
        {
            "trade_id": "trd_1234567890",
            "price": "30000",
            "quantity": "0.05",
            "amount": "1500",
            "fee": "0.75",
            "total": "1500.75",
            "role": "maker",
            "timestamp": "2024-01-15T10:35:00Z"
        }
    ],
    "average_price": "30000"
}
```

### Get All Orders

#### Endpoint
```
GET /orders
```

#### Query Parameters

| Parameter    | Type    | Description                            | Example          |
|--------------|---------|----------------------------------------|------------------|
| `page`       | number  | Page number (0-based)                  | `0`              |
| `page_size`  | number  | Results per page                       | `10`             |
| `ticker`     | string  | Filter by market pair                  | `BTC_USDT`       |
| `side`       | string  | Filter by side (comma-separated)       | `BUY,SELL`       |
| `order_type` | string  | Filter by order type (comma-separated) | `LIMIT,MARKET`   |
| `status`     | string  | Filter by status (comma-separated)     | `PENDING,FILLED` |
| `open`       | boolean | Get only open orders                   | `true`           |

#### Response Format
```javascript
{
    "page": 0,
    "page_size": 10,
    "total": 25,
    "data": [
        {
            "order_id": "ord_1234567890",
            "ticker": "BTC/USDT",
            "price": "30000",
            "quantity": "0.1",
            "remaining_quantity": "0.05",
            "side": "BUY",
            "order_type": "LIMIT",
            "status": "PARTIALLY_FILLED",
            "timestamp": "2024-01-15T10:30:00Z",
            "token": "BTC",
            "quote_asset": "USDT"
        }
        // ... more orders
    ]
}
```

---

## Order Cancellation

### Cancel Single Order

#### Endpoint
```
DELETE /orders/:orderId
```

Cancellation requires a wallet signature passed as **query parameters** (not in the request body).

#### Query Parameters

| Parameter   | Type   | Required | Description                    |
|-------------|--------|----------|--------------------------------|
| `order_id`  | string | Yes      | The order ID to cancel         |
| `timestamp` | number | Yes      | Integer milliseconds           |
| `signature` | string | Yes      | Wallet signature               |

#### Wallet Signing Payload
```javascript
const payload = {
    action: 'CANCEL_ORDER',
    order_id: 'ord_1234567890',
    timestamp: 1700000000000
};
const signingString = canonicalizePayload(payload);
const signature = await wallet.signMessage(signingString);
```

#### Example Request
```javascript
const wallet = ethers.Wallet.fromPhrase(WALLET_SEED);
const timestamp = Date.now();
const orderId = 'ord_1234567890';

// 1. Sign the cancel message
const cancelPayload = {
    action: 'CANCEL_ORDER',
    order_id: orderId,
    timestamp
};
const signingString = canonicalizePayload(cancelPayload);
const walletSignature = await wallet.signMessage(signingString);

// 2. Build query string
const queryParams = new URLSearchParams({
    order_id: orderId,
    timestamp: timestamp.toString(),
    signature: walletSignature
});

// 3. Generate HMAC signature
const method = 'DELETE';
const path = `/orders/${orderId}?${queryParams.toString()}`;
const hmacSignature = signRequest(method, path, {}, timestamp, API_SECRET);

// 4. Send request
const response = await fetch(
    `https://testnet-api.kalqix.com/v1/orders/${orderId}?${queryParams.toString()}`,
    {
        method: 'DELETE',
        headers: {
            'x-api-key': API_KEY,
            'x-api-signature': hmacSignature,
            'x-api-timestamp': timestamp.toString(),
            'Content-Type': 'application/json'
        }
    }
);
```

#### Response Format

**Success Response (200)**
```javascript
{
    "message": "Your cancellation request is received."
}
```

**Error Responses**

**400 Bad Request**
```javascript
{
    "error": "Invalid order ID."
}
```

**404 Not Found**
```javascript
{
    "error": "Order not found."
}
```

**400 Bad Request**
```javascript
{
    "error": "Your cancellation request is already received."
}
```

**400 Bad Request**
```javascript
{
    "error": "Order cannot be canceled or is already processed."
}
```

**403 Forbidden**
```javascript
{
    "error": "Permission Denied. Order is already filled."
}
```

---

## Cancel All Orders

### Endpoint
```
DELETE /users/me/cancel-all-orders
```

Cancels all open orders for a specific market. Requires a wallet signature passed as **query parameters**.

#### Query Parameters

| Parameter   | Type   | Required | Description                    |
|-------------|--------|----------|--------------------------------|
| `market`    | number | Yes      | Market ID (integer)            |
| `timestamp` | number | Yes      | Integer milliseconds           |
| `signature` | string | Yes      | Wallet signature               |

#### Wallet Signing Payload
```javascript
const payload = {
    action: 'CANCEL_ALL_ORDERS',
    market: 1,                    // Integer market ID
    timestamp: 1700000000000
};
const signingString = canonicalizePayload(payload);
const signature = await wallet.signMessage(signingString);
```

#### Example Request
```javascript
const wallet = ethers.Wallet.fromPhrase(WALLET_SEED);
const timestamp = Date.now();
const marketId = 1;

const cancelAllPayload = {
    action: 'CANCEL_ALL_ORDERS',
    market: marketId,
    timestamp
};
const signingString = canonicalizePayload(cancelAllPayload);
const walletSignature = await wallet.signMessage(signingString);

const queryParams = new URLSearchParams({
    market: marketId.toString(),
    timestamp: timestamp.toString(),
    signature: walletSignature
});

const method = 'DELETE';
const path = `/users/me/cancel-all-orders?${queryParams.toString()}`;
const hmacSignature = signRequest(method, path, {}, timestamp, API_SECRET);

const response = await fetch(
    `https://testnet-api.kalqix.com/v1/users/me/cancel-all-orders?${queryParams.toString()}`,
    {
        method: 'DELETE',
        headers: {
            'x-api-key': API_KEY,
            'x-api-signature': hmacSignature,
            'x-api-timestamp': timestamp.toString(),
            'Content-Type': 'application/json'
        }
    }
);
```

#### Response Format

**Success Response (200)**
```javascript
{
    "message": "Your cancel all request is received."
}
```

---

## Order Types and Status

### Order Types

| Type     | Description                            | Price Required |
|----------|----------------------------------------|----------------|
| `LIMIT`  | Order placed at a specific price       | Yes            |
| `MARKET` | Order executed at current market price | No             |

### Time in Force

| Value | Name | Description |
|-------|------|-------------|
| `0`   | GTC  | Good Till Cancel (default) |
| `1`   | IOC  | Immediate or Cancel |
| `2`   | FOK  | Fill or Kill |

### Order Statuses

| Status                   | Description                             |
|--------------------------|-----------------------------------------|
| `PENDING`                | Order is waiting to be filled           |
| `PARTIALLY_FILLED`       | Order has been partially filled         |
| `FILLED`                 | Order has been completely filled        |
| `CANCELLATION_REQUESTED` | Cancellation request has been submitted |
| `CANCELLED`              | Order has been cancelled                |

### Cancellation Rules

- Only `LIMIT` orders can be cancelled
- Only orders with status `PENDING` or `PARTIALLY_FILLED` can be cancelled
- Orders that are already filled cannot be cancelled
- Market orders cannot be cancelled (they execute immediately)

---

## Error Handling

### Common Error Codes

| Code | Description           | Solution                            |
|------|-----------------------|-------------------------------------|
| 400  | Bad Request           | Check request parameters and format |
| 401  | Unauthorized          | Verify API key and signature        |
| 403  | Forbidden             | Check permissions and order status  |
| 404  | Not Found             | Verify order ID or market exists    |
| 429  | Too Many Requests     | Respect rate limits                 |
| 500  | Internal Server Error | Contact support                     |

### Error Response Format
```javascript
{
    "error": "Error description message"
}
```

---

## Rate Limiting

- **Default:** 1000 requests/sec per IP
- **Headers:** `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- **On exceeding:** HTTP 429 with `Retry-After` header
