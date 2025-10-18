# Overview

Boss, this **Billstack** setup will be walking you to:
- Generate virtual bank accounts tied to users
- Store account details in a MongoDB collection
- Handle incoming webhooks from Billstack (like receiving a credit alert via API POST from Billstack)

---

## What You Need to Setup

- **Billstack Account:** Sign up at [billstack.co](https://billstack.co) to get an API key.  
- **Model Creation:** Create a `virtualAccount.js` model for storing user virtual accounts.  
- **API Secret Key:** Get it from the **BILLSTACK Dashboard → Developer API** sidebar menu.

---

### Add The Billstack credentials into Environment Variables

```env
BILLSTACK_BASE_URL='https://api.billstack.co/v2'  # Billstack base URL endpoint
BILLSTACK_API_KEY=***********                     # Billstack API key
```

---

### Here is a Sample of the Virtual Account Model

```js
const mongoose = require('mongoose');

const virtualAccountSchema = new mongoose.Schema({
  userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  accountNumber: { type: String, required: true },
  accountName: { type: String, required: true },
  bankName: { type: String, required: true },
  accountReference: { type: String, required: true },
  createdAt: { type: Date, required: true }
});

module.exports = mongoose.model('VirtualAccount', virtualAccountSchema);
```

---

### User Model Requirements

Make sure say the **User model** has these field included:
- `email`
- `phoneNumber`
- `firstName`
- `lastName`

> Billstack needs this data to generate the account.

---

### Utility Function to Generate Unique Bank Reference

Create file: `utils/generateRef.js`

```js
const generateBankReference = () => {
  return `REF-${Date.now()}-${Math.random().toString(36).substring(2, 8).toUpperCase()}`;
};
module.exports = generateBankReference;
```

Billstack requires a **unique reference string** while generating an account.

---

## Core Function

Create a file `services/billstackService.js` to define the logic for generating a virtual account for a user (if none exists).

### Example Logic

```js
const axios = require('axios');
const generateBankReference = require('../utils/generateRef'); // Import the reference function way we created
const VirtualAccount = require('../models/VirtualAccount');
const logger = require('../utils/logger');

const createVirtualAccount = async (userId) => {
  try {
    const existing = await VirtualAccount.findOne({ userId });
    if (existing) return existing;

    const User = require('../models/User');  // Import dynamically if needed
    const user = await User.findById(userId);
    if (!user) throw new Error('User not found');

    const { email, phoneNumber, firstName, lastName } = user;

    // Generate random account from random Bank
    // You can limit the account to a specific bank if you wish
    // const bank = 'PALMPAY'

    const banks = ['9PSB', 'SAFEHAVEN', 'PROVIDUS', 'BANKLY', 'PALMPAY'];
    const bank = banks[Math.floor(Math.random() * banks.length)];
    const reference = generateBankReference();

    const payload = {
      reference,
      email,
      phone: phoneNumber,
      firstName,
      lastName,
      bank,
    };

    const response = await axios.post(
      `${process.env.BILLSTACK_BASE_URL}/thirdparty/generateVirtualAccount/`,
      payload,
      {
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${process.env.BILLSTACK_API_KEY}`,
        },
      }
    );

    if (!response.data.status || !response.data.data.account[0]) {
      throw new Error(response.data.message || 'Failed to reserve virtual account');
    }

    const account = response.data.data.account[0];

    // Save account details in database
    const virtualAccount = await VirtualAccount.create({
      userId,
      accountNumber: account.account_number,
      accountName: account.account_name,
      bankName: account.bank_name,
      accountReference: response.data.data.reference,
      createdAt: new Date(account.created_at),
    });

    logger.info(`Virtual account created for user ${userId}: ${account.account_number}`);
    return virtualAccount;
  } catch (error) {
    logger.error(`Billstack account creation failed for user ${userId}: ${error.message}`);
    throw new Error('Virtual account creation failed');
  }
};
```

---

### Function to Fetch or Create Account

```js
const getVirtualAccount = async (userId) => {
  return (await VirtualAccount.findOne({ userId })) || (await createVirtualAccount(userId));
};
```

> It creates a new virtual account if none exists, or fetches an existing one.

---

## Webhook Handler

This shows how you fit handle webhook request (Payment alert) when Billstack notify you say this user don transfer him payment

- Create a route for the webhook, e.g., `POST /api/webhooks/billstack`
- In your controller file, define the webhook logic:

```js
const crypto = require('crypto');
const logger = require('../utils/logger');
const VirtualAccount = require('../models/VirtualAccount');
const Transaction = require('../models/Transaction');

const handleBillstackWebhook = async (req, res) => {
  try {
    const WEBHOOK_SECRET = process.env.BILLSTACK_API_KEY;

    // Verify incoming webhook request
    const headerSig = req.get('x-wiaxy-signature');
    if (!headerSig) {
      logger.error('Missing signature header');
      return res.status(400).send('Missing signature');
    }

    const expected = crypto.createHash('md5').update(WEBHOOK_SECRET).digest('hex');

    if (headerSig !== expected) {
      logger.error(`Invalid signature. Got: ${headerSig}, Expected: ${expected}`);
      return res.status(401).send('Invalid signature');
    }

    const payload = req.body;

    if (!payload.event || !payload.data) {
      logger.error('Bad payload', payload);
      return res.status(400).send('Bad payload');
    }

    if (payload.event === 'PAYMENT_NOTIFICATION' || payload.event === 'PAYMENT_NOTIFIFICATION') {
      const dt = payload.data;

      if (dt.type === 'RESERVED_ACCOUNT_TRANSACTION') {
        const amount = parseFloat(dt.amount);
        const reference = dt.reference;
        const accountNumber = dt.account?.account_number;

        logger.info(`Received payment: ${JSON.stringify(dt)}`);

        // Check if the user account exists
        const virtualAccount = await VirtualAccount.findOne({ accountNumber });
        if (!virtualAccount) {
          logger.error(`No virtual account found for account number: ${accountNumber}`);
          return res.status(400).json({ error: 'Virtual account not found' });
        }

        // Perform actions such as crediting the user wallet balance here

        return res.status(200).json({ status: 'success' });
      }
    }
  } catch (error) {
    logger.error(`Webhook processing failed: ${error.message}`);
    res.status(500).json({ error: 'Webhook processing failed' });
  }
};
```

---

### Webhook Setup Notes

Before the webhook works:
- You go need expose the webhook endpoint publicly. (for example, https://railway-server-url.com/api/webhook)
- Add your webhook URL in the **Billstack Dashboard → Developer API → Webhook URL** input.

> Billstack notifies you of events within your account via the webhook URL you provide.

---

### Sample Payload from Billstack

*This is a sample of what Billstack will send to the submitted endpoint (webhook) when user transfer money into the account*

```json
{
  "event": "PAYMENT_NOTIFIFICATION",
  "data": {
    "type": "RESERVED_ACCOUNT_TRANSACTION",
    "reference": "transaction_reference",
    "merchant_reference": "account_reference",
    "wiaxy_ref": "inter_bank_reference",
    "amount": "payment_amount",
    "created_at": "receiving_date",
    "account": {
      "account_number": "customer_account_number",
      "account_name": "customer_account_name",
      "bank_name": "customer_bank_name",
      "created_at": "account_creation_date"
    },
    "payer": {
      "account_number": "payer_acct_num",
      "first_name": "payer_first_name",
      "last_name": "payer_last_name",
      "createdAt": "payment_date"
    }
  }
}
```

You can use this payload to verify the sender and perform actions such as topping up the user’s wallet balance.

