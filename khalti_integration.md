# Khalti Payment Gateway Integration — Django REST Framework

> A complete, production-ready guide to integrating Khalti's epayment API (v2) into a Django/DRF backend for Nepali e-commerce applications.
### By Prashant Maharjan
---

## Table of Contents

1. [How Khalti Actually Works](#1-how-khalti-actually-works)
2. [The Complete Payment Lifecycle](#2-the-complete-payment-lifecycle)
3. [Khalti vs eSewa — Architectural Difference](#3-khalti-vs-esewa--architectural-difference)
4. [Prerequisites](#4-prerequisites)
5. [Step 1 — Django Settings Configuration](#step-1--django-settings-configuration)
6. [Step 2 — The Order Model](#step-2--the-order-model)
7. [Step 3 — Payment Strategy (Initiating Payment)](#step-3--payment-strategy-initiating-payment)
8. [Step 4 — The Order Creation View](#step-4--the-order-creation-view)
9. [Step 5 — The Verification Callback View](#step-5--the-verification-callback-view)
10. [Step 6 — URL Configuration](#step-6--url-configuration)
11. [Step 7 — Frontend Integration](#step-7--frontend-integration)
12. [Testing with Sandbox Credentials](#testing-with-sandbox-credentials)
13. [Production Checklist](#production-checklist)
14. [Common Pitfalls & Debugging](#common-pitfalls--debugging)

---

## 1. How Khalti Actually Works

Khalti is Nepal's developer-friendly digital payment gateway. Unlike eSewa (which uses a form-submission redirect), Khalti is **API-first**: your Django backend talks directly to Khalti's server via HTTP REST APIs.

### The Mental Model

Think of Khalti as a two-step handshake:

1. **Step 1 — Your server tells Khalti:** "I have an order for ₨1500. Here are the details. Give me a payment page URL."
2. **Step 2 — After user pays, your server asks Khalti:** "Did payment `pidx=ABC123` actually go through? What's the real status?"

This is fundamentally different from eSewa. With eSewa, the browser does the work. With Khalti, your **server** initiates and verifies the payment directly.

### Key Concepts

| Concept | Description |
|---|---|
| **`pidx`** | Payment IDX — Khalti's unique identifier for a payment session. Think of it as a "ticket number" for the transaction. |
| **`Secret Key`** | Your server-side API key. **Never expose this on the frontend.** It authenticates your server to Khalti. |
| **`return_url`** | Where Khalti redirects the user's browser after they pay (or cancel). |
| **`website_url`** | Your website's base URL. Used by Khalti for branding/display purposes. |
| **Amount in Paisa** | Khalti accepts amounts in **paisa** (1/100th of a rupee). So ₨1500 = `150000` paisa. |

---

## 2. The Complete Payment Lifecycle

```
┌─────────┐         ┌──────────┐         ┌──────────┐         ┌──────────┐
│ Frontend │         │  Django  │         │  Khalti  │         │  Django  │
│  (React) │         │ Backend  │         │ Servers  │         │ Callback │
└────┬─────┘         └────┬─────┘         └────┬─────┘         └────┬─────┘
     │                    │                    │                    │
     │  1. POST /orders/  │                    │                    │
     │  {payment: "Khalti"}                    │                    │
     │───────────────────>│                    │                    │
     │                    │                    │                    │
     │                    │ 2. Create Order    │                    │
     │                    │                    │                    │
     │                    │ 3. POST /epayment/initiate/            │
     │                    │    {amount, order_id, return_url}       │
     │                    │───────────────────>│                    │
     │                    │                    │                    │
     │                    │ 4. Response:       │                    │
     │                    │    {pidx, payment_url}                  │
     │                    │<───────────────────│                    │
     │                    │                    │                    │
     │  5. Return         │                    │                    │
     │     {pidx,         │                    │                    │
     │      payment_url}  │                    │                    │
     │<───────────────────│                    │                    │
     │                    │                    │                    │
     │  6. window.location = payment_url       │                    │
     │────────────────────────────────────────>│                    │
     │                    │                    │                    │
     │                    │                    │ 7. User enters     │
     │                    │                    │    PIN and pays    │
     │                    │                    │                    │
     │  8. Redirect to return_url?pidx=...&status=Completed        │
     │<───────────────────────────────────────│                    │
     │                    │                    │                    │
     │  9. Forward to callback                 │                    │
     │─────────────────────────────────────────────────────────────>│
     │                    │                    │                    │
     │                    │                    │  10. POST /lookup/ │
     │                    │                    │      {pidx}        │
     │                    │                    │<────────────────────│
     │                    │                    │                    │
     │                    │                    │  11. {status:      │
     │                    │                    │   "Completed",...} │
     │                    │                    │─────────────────────>│
     │                    │                    │                    │
     │                    │                    │  12. Update order  │
     │                    │                    │                    │
     │  13. "Payment successful"                                   │
     │<────────────────────────────────────────────────────────────│
```

### Step-by-step breakdown:

1. **User places order** — Frontend sends `POST /api/orders/` with `payment_method: "Khalti"`.
2. **Backend creates the order** — Django creates the Order record in the database.
3. **Backend calls Khalti's initiate API** — Your Django server makes a `POST` request to `https://dev.khalti.com/api/v2/epayment/initiate/` with the order details. This is a **server-to-server** call — the user's browser is not involved.
4. **Khalti returns `pidx` and `payment_url`** — `pidx` is a unique identifier for this payment session. `payment_url` is a Khalti-hosted page where the user will enter their PIN.
5. **Backend returns both to the frontend** — The frontend gets the `payment_url` to redirect to and the `pidx` for reference.
6. **Frontend redirects the user** — `window.location.href = payment_url` takes the user to Khalti's payment page.
7. **User completes payment on Khalti** — They enter their Khalti PIN and confirm the payment.
8. **Khalti redirects back** — The user's browser is redirected to your `return_url` with `?pidx=...&status=Completed` in the query params.
9. **The callback endpoint receives the request** — Your Django view picks up the `pidx` and `status`.
10. **Server-side verification (CRITICAL)** — Your backend makes a `POST` request to Khalti's lookup API to verify the payment. **Never trust the query params alone** — they can be spoofed.
11. **Khalti confirms the payment** — Returns the real status, amount, and order details.
12. **Order is updated** — Mark `payment_status = "Completed"` and store the `pidx` as `transaction_id`.
13. **User sees confirmation** — The frontend renders a success page.

---

## 3. Khalti vs eSewa — Architectural Difference

Understanding this distinction is important for your code architecture:

| Aspect | eSewa | Khalti |
|---|---|---|
| **Payment initiation** | Frontend submits a form to eSewa | Backend calls Khalti's API |
| **Signature/Auth** | HMAC-SHA256 signature in form | Secret Key in `Authorization` header |
| **Server communication** | None during initiation | Direct HTTP POST to Khalti |
| **Amount format** | String with 2 decimals (`"1500.00"`) | Integer in **paisa** (`150000`) |
| **Verification** | Decode Base64 from callback | POST to Khalti's lookup endpoint |
| **Security model** | Signature-based | API key-based |

**Bottom line:** Khalti is more straightforward for developers because it follows standard REST API patterns. But it requires your server to be able to make outbound HTTP requests (which eSewa doesn't).

---

## 4. Prerequisites

Before you begin:

- **Python 3.10+** and **Django 4.2+** (or 5.x/6.x)
- **Django REST Framework** installed
- **`requests` library** installed — for making HTTP calls to Khalti's API
- **A Khalti Merchant Account** — Get test credentials from [merchant.khalti.com](https://merchant.khalti.com)

```bash
pip install djangorestframework requests
```

---

## Step 1 — Django Settings Configuration

```python
# settings.py

KHALTI_SETTINGS = {
    # Khalti epayment v2 endpoints
    "INITIATE_URL": "https://dev.khalti.com/api/v2/epayment/initiate/",     # Sandbox
    "LOOKUP_URL": "https://dev.khalti.com/api/v2/epayment/lookup/",         # Sandbox
    
    # Your credentials — get these from the Khalti merchant dashboard
    "SECRET_KEY": "test_secret_key_your_key_here",
    
    # Where Khalti redirects after payment (your Django callback)
    "SUCCESS_URL": "http://localhost:8000/api/orders/khalti-verify/",
    
    # Your website URL (used by Khalti for branding)
    "WEBSITE_URL": "http://localhost:3000",
}
```

**For production**, change:
- `INITIATE_URL` → `"https://khalti.com/api/v2/epayment/initiate/"`
- `LOOKUP_URL` → `"https://khalti.com/api/v2/epayment/lookup/"`
- `SECRET_KEY` → Your live secret key from the Khalti dashboard
- All URLs → HTTPS production domains

> **Important:** The `SECRET_KEY` is your **server-side** key (starts with `test_secret_key_` for sandbox or `live_secret_key_` for production). Never use the public key here.

---

## Step 2 — The Order Model

Khalti uses the same Order model as other gateways. The key fields:

```python
# orders/models.py

from django.db import models
from django.conf import settings

class Order(models.Model):
    PAYMENT_METHODS = (
        ('eSewa', 'eSewa'),
        ('Khalti', 'Khalti'),
        ('COD', 'Cash on Delivery'),
    )

    PAYMENT_STATUS = (
        ('Pending', 'Pending'),
        ('Completed', 'Completed'),
        ('Failed', 'Failed'),
    )

    order_code = models.CharField(max_length=10, unique=True)
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    
    payment_method = models.CharField(max_length=50, choices=PAYMENT_METHODS)
    payment_status = models.CharField(max_length=20, choices=PAYMENT_STATUS, default='Pending')
    
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    
    # Khalti-specific: store the `pidx` returned from initiation
    transaction_id = models.CharField(max_length=100, null=True, blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)
```

For Khalti, `transaction_id` stores the `pidx` — first during initiation (before the user pays) and then confirmed after verification.

---

## Step 3 — Payment Strategy (Initiating Payment)

This is where Khalti differs from eSewa. Instead of building a form, we make a **direct HTTP POST** to Khalti's server:

```python
# orders/payment_strategies.py

import requests
from django.conf import settings

class KhaltiStrategy:
    """
    Initiates a Khalti payment session.
    
    Unlike eSewa (which is form-based), Khalti requires a server-to-server
    API call. We POST the order details to Khalti, and they return a
    payment URL where the user can complete the payment.
    """
    
    def get_payment_payload(self, order):
        conf = settings.KHALTI_SETTINGS
        
        # CRITICAL: Khalti expects amount in PAISA (integer)
        # Rs. 1500.00 = 150000 paisa
        amount_paisa = int(order.total_amount * 100)

        # Authentication: Khalti uses API key in the Authorization header
        headers = {
            'Authorization': f"Key {conf['SECRET_KEY']}",
            'Content-Type': 'application/json',
        }

        # The payload tells Khalti about the order
        payload = {
            "return_url": conf["SUCCESS_URL"],      # Where to redirect after payment
            "website_url": conf["WEBSITE_URL"],      # Your website (for branding)
            "amount": amount_paisa,                  # Amount in PAISA
            "purchase_order_id": order.order_code,   # Your unique order reference
            "purchase_order_name": f"Order {order.order_code}",
            "customer_info": {
                "name": order.user.get_full_name() or order.user.username,
                "email": order.user.email,
                "phone": str(getattr(order.user, 'phone', '9800000000'))
            }
        }

        try:
            # Make the server-to-server call
            response = requests.post(
                conf["INITIATE_URL"],
                json=payload,
                headers=headers,
                timeout=15  # Don't hang forever if Khalti is slow
            )
            data = response.json()

            if response.status_code == 200:
                """
                Successful response looks like:
                {
                    "pidx": "bZQLD9wRVWo4CdESSfeMSU",
                    "payment_url": "https://test-pay.khalti.com/?pidx=bZQLD9wRVWo4CdESSfeMSU",
                    "expires_at": "2025-04-16T12:00:00+05:45",
                    "expires_in": 1800
                }
                """
                return {
                    "payment_method": "Khalti",
                    "pidx": data.get("pidx"),           # Save this to the order
                    "payment_url": data.get("payment_url"),  # Redirect user here
                }
            
            # Khalti returned an error
            return {"error": "Khalti initiation failed", "details": data}

        except requests.exceptions.RequestException as e:
            return {"error": f"Khalti server connection failed: {str(e)}"}
```

### What happens during initiation:

```
Your Django Server                          Khalti Server
     │                                           │
     │  POST /api/v2/epayment/initiate/          │
     │  Headers: Authorization: Key test_...     │
     │  Body: {                                  │
     │    "return_url": "...",                    │
     │    "amount": 150000,                      │
     │    "purchase_order_id": "A1B2C3"          │
     │  }                                        │
     │──────────────────────────────────────────>│
     │                                           │
     │  200 OK                                   │
     │  {                                        │
     │    "pidx": "bZQLD...",                    │
     │    "payment_url": "https://test-pay..."   │
     │  }                                        │
     │<──────────────────────────────────────────│
```

### Key points about the amount:

```python
# ✅ Correct — Integer paisa
amount_paisa = int(order.total_amount * 100)  # Rs. 1500.50 → 150050

# ❌ Wrong — Khalti will reject floats
amount = float(order.total_amount * 100)  # 150050.0 — not integer!

# ❌ Wrong — Khalti will reject rupees
amount = order.total_amount  # 1500.50 — this is rupees, not paisa!
```

---

## Step 4 — The Order Creation View

When creating an order with Khalti, save the `pidx` to the order immediately:

```python
# orders/views.py

from rest_framework import viewsets, status
from rest_framework.response import Response
from .models import Order
from .payment_strategies import KhaltiStrategy

class OrderViewSet(viewsets.ModelViewSet):
    
    def create(self, request, *args, **kwargs):
        # ... your order creation logic ...
        order = ...  # Your newly created Order instance
        
        if order.payment_method == 'Khalti':
            strategy = KhaltiStrategy()
            payment_data = strategy.get_payment_payload(order)
            
            # Save the pidx to the order so we can match it later
            if 'pidx' in payment_data:
                order.transaction_id = payment_data['pidx']
                order.save()
            
            return Response({
                "message": "Order initiated. Proceed to payment.",
                **payment_data
            }, status=status.HTTP_201_CREATED)
```

### What the API response looks like:

```json
{
    "message": "Order initiated. Proceed to payment.",
    "payment_method": "Khalti",
    "pidx": "bZQLD9wRVWo4CdESSfeMSU",
    "payment_url": "https://test-pay.khalti.com/?pidx=bZQLD9wRVWo4CdESSfeMSU"
}
```

---

## Step 5 — The Verification Callback View

This is the **most security-critical** part. After the user pays, Khalti redirects them to your `return_url` with the `pidx` and `status` in the URL. **But you must NOT trust these query params.** An attacker could craft a URL like:

```
/khalti-verify/?pidx=FAKE&status=Completed
```

You **must** verify with Khalti's server directly:

```python
# orders/views.py

import requests
from django.conf import settings
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from django.shortcuts import get_object_or_404
from .models import Order

class PaymentCallbackViewSet(viewsets.ViewSet):
    
    @action(detail=False, methods=['get'], url_path='khalti-verify')
    def khalti_verify(self, request):
        """
        Khalti redirects here after payment:
        GET /api/orders/khalti-verify/?pidx=bZQLD...&status=Completed&...
        
        SECURITY: We do NOT trust the query params. We verify with
        Khalti's server using the lookup API.
        """
        pidx = request.query_params.get('pidx')
        status_received = request.query_params.get('status')
        
        # Quick sanity check before hitting Khalti's API
        if not pidx or status_received != "Completed":
            return Response({"error": "Payment not completed or pidx missing"}, status=400)

        # === SERVER-SIDE VERIFICATION ===
        # This is the security-critical part.
        # We ask Khalti directly: "Is this payment real?"
        
        conf = settings.KHALTI_SETTINGS
        headers = {'Authorization': f"Key {conf['SECRET_KEY']}"}
        
        verify_response = requests.post(
            conf["LOOKUP_URL"],
            json={"pidx": pidx},
            headers=headers
        )
        res_data = verify_response.json()

        """
        Khalti lookup response looks like:
        {
            "pidx": "bZQLD9wRVWo4CdESSfeMSU",
            "total_amount": 150000,                    <- in paisa
            "status": "Completed",                     <- "Completed", "Pending", "Initiated", etc.
            "transaction_id": "4H7AhoXDJWg5RvYBMGn5iY",
            "fee": 0,
            "refunded": false,
            "purchase_order_id": "A1B2C3",             <- Your order_code
            "purchase_order_name": "Order A1B2C3",
            "extra_merchant_params": null
        }
        """

        if res_data.get('status') == "Completed":
            # Find and update the order
            order_code = res_data.get('purchase_order_id')
            order = get_object_or_404(Order, order_code=order_code)
            
            # Mark as paid
            order.payment_status = "Completed"
            order.transaction_id = pidx
            order.save()
            
            return Response({
                "message": "Payment successful",
                "order_code": order.order_code
            })

        return Response({"error": "Verification failed with Khalti server"}, status=400)
```

### Why server-side verification matters:

```
                  ❌ INSECURE (Never do this)
                  ─────────────────────────
                  Just check query params:
                  if status == "Completed":
                      order.payment_status = "Completed"
                  
                  Anyone can fake: /khalti-verify/?status=Completed&pidx=anything
                  

                  ✅ SECURE (Always do this)
                  ─────────────────────────
                  Ask Khalti's server:
                  POST /api/v2/epayment/lookup/
                  Authorization: Key your_secret
                  Body: {"pidx": "the_real_pidx"}
                  
                  Only Khalti can confirm if the payment actually went through.
```

---

## Step 6 — URL Configuration

```python
# orders/urls.py

from django.urls import path, include
from rest_framework.routers import SimpleRouter
from .views import OrderViewSet, PaymentCallbackViewSet

router = SimpleRouter()
router.register(r'', OrderViewSet, basename='order')

urlpatterns = [
    path('khalti-verify/', PaymentCallbackViewSet.as_view({'get': 'khalti_verify'}), name='khalti-verify'),
    path('', include(router.urls)),
]
```

Main `urls.py`:

```python
# project/urls.py

urlpatterns = [
    path('api/orders/', include('orders.urls')),
]
```

This gives you: `GET /api/orders/khalti-verify/` — which must match the `SUCCESS_URL` in your settings.

---

## Step 7 — Frontend Integration

Khalti is simpler on the frontend than eSewa — you just redirect to the `payment_url`:

```jsx
// React component

const placeOrder = async () => {
    const response = await fetch('/api/orders/', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Authorization': `Bearer ${token}`
        },
        body: JSON.stringify({
            address_id: 5,
            payment_method: 'Khalti'
        })
    });
    
    const data = await response.json();
    
    if (data.payment_method === 'Khalti' && data.payment_url) {
        // Simply redirect — no form submission needed
        window.location.href = data.payment_url;
    }
    
    if (data.error) {
        alert(`Payment failed: ${data.error}`);
    }
};
```

### What the user sees:

1. User clicks "Pay with Khalti" → Browser navigates to `https://test-pay.khalti.com/?pidx=...`
2. Khalti shows their payment page → User enters their Khalti PIN
3. After payment → Browser redirects to `http://localhost:8000/api/orders/khalti-verify/?pidx=...&status=Completed`
4. Django verifies and redirects to a success page

---

## Testing with Sandbox Credentials

### Sandbox endpoints:

| Setting | Sandbox Value |
|---|---|
| Initiate URL | `https://dev.khalti.com/api/v2/epayment/initiate/` |
| Lookup URL | `https://dev.khalti.com/api/v2/epayment/lookup/` |
| Payment page | `https://test-pay.khalti.com/` |

### Getting test credentials:

1. Go to [merchant.khalti.com](https://merchant.khalti.com)
2. Create a merchant account (or use the demo one from their docs)
3. Find your **Test Secret Key** in the dashboard (starts with `test_secret_key_`)

### Sandbox test accounts:

Khalti provides test customer accounts. Check their [documentation](https://docs.khalti.com/khalti-epayment/) for the latest test credentials (MPIN, phone number, OTP).

### Quick test command:

```bash
# Test the initiate API directly using curl
curl -X POST "https://dev.khalti.com/api/v2/epayment/initiate/" \
  -H "Authorization: Key test_secret_key_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "return_url": "http://localhost:8000/api/orders/khalti-verify/",
    "website_url": "http://localhost:3000",
    "amount": 1000,
    "purchase_order_id": "TEST001",
    "purchase_order_name": "Test Order"
  }'
```

Expected response:
```json
{
    "pidx": "bZQLD9wRVWo4CdESSfeMSU",
    "payment_url": "https://test-pay.khalti.com/?pidx=bZQLD9wRVWo4CdESSfeMSU",
    "expires_at": "2025-04-16T12:00:00+05:45",
    "expires_in": 1800
}
```

---

## Production Checklist

Before going live:

- [ ] Replace sandbox credentials with live credentials from the Khalti merchant dashboard
- [ ] Change `INITIATE_URL` to `"https://khalti.com/api/v2/epayment/initiate/"`
- [ ] Change `LOOKUP_URL` to `"https://khalti.com/api/v2/epayment/lookup/"`
- [ ] Update `SUCCESS_URL` to use HTTPS production domain
- [ ] Move `SECRET_KEY` to environment variables
- [ ] Add amount verification: compare `res_data['total_amount']` with your order's amount during lookup
- [ ] Add idempotency: don't process the same `pidx` twice
- [ ] Set appropriate timeout on HTTP requests (15-30 seconds)
- [ ] Handle Khalti server downtime gracefully
- [ ] Log all initiation and verification requests for auditing

---

## Common Pitfalls & Debugging

### 1. "Amount should be in paisa" / 400 error on initiate

**Cause:** You're sending the amount in rupees instead of paisa.

```python
# ❌ Wrong
amount = order.total_amount  # 1500.00 (rupees)

# ✅ Correct
amount = int(order.total_amount * 100)  # 150000 (paisa)
```

### 2. "Unauthorized" / 401 error on initiate

**Cause:** Wrong API key format or using the public key instead of the secret key.

```python
# ❌ Wrong — using public key
headers = {'Authorization': f"Key test_public_key_..."}

# ❌ Wrong — missing "Key " prefix
headers = {'Authorization': f"test_secret_key_..."}

# ✅ Correct
headers = {'Authorization': f"Key test_secret_key_..."}
```

### 3. Payment succeeds but verification fails

**Cause:** Using sandbox credentials for initiation but production URL for lookup (or vice versa).

Make sure both `INITIATE_URL` and `LOOKUP_URL` point to the same environment (both `dev.khalti.com` or both `khalti.com`).

### 4. `pidx` not found during verification

**Cause:** The `pidx` in the callback doesn't match what Khalti sent. This can happen if:
- You're testing with an old/expired payment session
- The callback URL was manually constructed (potential attack)

Always verify with `purchase_order_id` from the lookup response, not from the callback params.

### 5. Order amount mismatch (security issue)

**Problem:** An attacker could initiate a session for ₨1 but your order is ₨1500.

**Solution:** Add amount verification in your callback:

```python
# Extra security: verify the amount matches
khalti_amount = res_data.get('total_amount')  # in paisa
order_amount_paisa = int(order.total_amount * 100)

if khalti_amount != order_amount_paisa:
    return Response({"error": "Amount mismatch — possible tampering"}, status=400)
```

### 6. Handling Khalti server timeouts

**Problem:** If Khalti's server is slow, your order creation API hangs.

**Solution:** Set a timeout and handle it gracefully:

```python
try:
    response = requests.post(url, json=payload, headers=headers, timeout=15)
except requests.exceptions.Timeout:
    return {"error": "Khalti server is not responding. Please try again."}
except requests.exceptions.ConnectionError:
    return {"error": "Cannot reach Khalti servers. Please check your connection."}
```

---

*Last updated: April 2026. Based on Khalti epayment API v2.*
