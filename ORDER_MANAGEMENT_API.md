# Order Management API Documentation

This document provides comprehensive guidance for managing orders through the Kalqix API, including placement, cancellation, and status retrieval.

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

## Table of Contents

1. [Authentication](#authentication)
2. [Order Placement](#order-placement)
3. [Order Status Retrieval](#order-status-retrieval)
4. [Order Cancellation](#order-cancellation)
5. [Order Types and Statuses](#order-types-and-status)
6. [Error Handling](#error-handling)
7. [Rate Limiting](#rate-limiting)
8. [Examples](#examples)

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
    const canonical = `${method}|${path}|${JSON.stringify(body)}|${timestamp}`;
    return crypto.createHmac('sha256', apiSecret).update(canonical).digest('hex');
}
```

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
    "message": "BUY 0.1 BTC_USDT @PRICE: 30000",  // Required: message
    "signature": "0x..."            // Required: Ethereum signature for message
}
```

### Ethereum Message Signing

Orders require Ethereum message signatures for security. Use your wallet seed to sign the order message:

```javascript
const { ethers } = require('ethers');

async function generateOrderMessage(order) {
    const price = order.order_type === 'LIMIT' 
        ? order.price 
        : 'MARKET';
    return `${order.side} ${order.quantity} ${order.ticker} @PRICE: ${price}`;
}

async function signOrderMessage(order, walletSeed) {
    const wallet = ethers.Wallet.fromPhrase(walletSeed);
    const message = await generateOrderMessage(order);
    const signature = await wallet.signMessage(message);
    
    return { message, signature, walletAddress: wallet.address };
}
```

### Complete Order Placement Example

```javascript
const crypto = require('crypto');
const { ethers } = require('ethers');

// Configuration
const API_KEY = process.env.API_KEY;
const API_SECRET = process.env.API_SECRET;
const WALLET_SEED = process.env.WALLET_SEED;

async function placeOrder(orderData) {
    // 1. Sign the order message
    const { message, signature } = await signOrderMessage(orderData, WALLET_SEED);
    
    // 2. Prepare request body
    const requestBody = {
        ...orderData,
        message,
        signature
    };
    
    // 3. Generate HMAC signature
    const method = 'POST';
    const path = '/orders';
    const timestamp = Date.now();
    const hmacSignature = signRequest(method, path, requestBody, timestamp, API_SECRET);
    
    // 4. Send request
    const response = await fetch('https://devapi.kalqix.com/v1/orders', {
        method: 'POST',
        headers: {
            'x-api-key': API_KEY,
            'x-api-signature': hmacSignature,
            'x-api-timestamp': timestamp.toString(),
            'Content-Type': 'application/json'
        },
        body: requestBody
    });
    
    return await response.json();
}

// Example usage
const order = {
    ticker: 'BTC_USDT',
    price: '30000',
    quantity: '0.1',
    side: 'BUY',
    order_type: 'LIMIT'
};

const result = await placeOrder(order);
console.log('Order placed:', result);
```

### Response Format

#### Success Response (200)
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

const response = await fetch(`https://devapi.kalqix.com/v1/orders/ord_1234567890`, {
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
| `ticker`     | string  | Filter by market pair                  | `BTC_USDT`       |
| `side`       | string  | Filter by side (comma-separated)       | `BUY,SELL`       |
| `order_type` | string  | Filter by order type (comma-separated) | `LIMIT,MARKET`   |
| `status`     | string  | Filter by status (comma-separated)     | `PENDING,FILLED` |
| `open`       | boolean | Get only open orders                   | `true`           |

#### Example Request
```javascript
const method = 'GET';
const path = '/orders?page=0&ticker=BTC_USDT&open=true';
const body = {};
const timestamp = Date.now();
const signature = signRequest(method, path, body, timestamp, apiSecret);

const response = await fetch('https://devapi.kalqix.com/v1/orders?page=0&ticker=BTC_USDT&open=true', {
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
    "page": 0,
    "pageSize": 10,
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

#### Example Request
```javascript
const method = 'DELETE';
const path = '/orders/ord_1234567890';
const body = {};
const timestamp = Date.now();
const signature = signRequest(method, path, body, timestamp, apiSecret);

const response = await fetch('https://devapi.kalqix.com/v1/orders/ord_1234567890', {
    method: 'DELETE',
    headers: {
        'x-api-key': apiKey,
        'x-api-signature': signature,
        'x-api-timestamp': timestamp.toString(),
        'Content-Type': 'application/json'
    }
});
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

## Order Types and Status

### Order Types

| Type     | Description                            | Price Required |
|----------|----------------------------------------|----------------|
| `LIMIT`  | Order placed at a specific price       | Yes            |
| `MARKET` | Order executed at current market price | No             |

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
