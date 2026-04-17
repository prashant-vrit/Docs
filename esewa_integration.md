# eSewa Payment Gateway Integration — Django REST Framework

> A complete, production-ready guide to integrating eSewa's epay v2 API into a Django/DRF backend for Nepali e-commerce applications.
### By Prashant Maharjan
---

## Table of Contents

1. [How eSewa Actually Works](#1-how-esewa-actually-works)
2. [The Complete Payment Lifecycle](#2-the-complete-payment-lifecycle)
3. [Understanding HMAC-SHA256 Signatures](#3-understanding-hmac-sha256-signatures)
4. [Prerequisites](#4-prerequisites)
5. [Step 1 — Django Settings Configuration](#step-1--django-settings-configuration)
6. [Step 2 — The Order Model](#step-2--the-order-model)
7. [Step 3 — Signature Generation Utility](#step-3--signature-generation-utility)
8. [Step 4 — Payment Strategy (Building the Payload)](#step-4--payment-strategy-building-the-payload)
9. [Step 5 — The Order Creation View](#step-5--the-order-creation-view)
10. [Step 6 — The Verification Callback View](#step-6--the-verification-callback-view)
11. [Step 7 — URL Configuration](#step-7--url-configuration)
12. [Step 8 — Frontend Integration](#step-8--frontend-integration)
13. [Testing with Sandbox Credentials](#testing-with-sandbox-credentials)
14. [Production Checklist](#production-checklist)
15. [Common Pitfalls & Debugging](#common-pitfalls--debugging)

---

## 1. How eSewa Actually Works

eSewa is Nepal's most widely used digital wallet. Unlike Khalti (which is API-first), eSewa uses a **form-submission redirect model**. This means your backend never directly talks to eSewa's server — instead, you build an HTML form payload on your server, send it to your frontend, and the frontend submits that form to eSewa's page.

### The Mental Model

Think of it like writing a cheque. Your backend fills out the cheque (the form payload) with the amount, order reference, and a tamper-proof seal (the HMAC signature). The frontend hands that cheque to eSewa. eSewa cashes it, then redirects the customer back to your website with a receipt (encoded in Base64).

Your backend then reads the receipt to confirm the payment actually went through.

**Key characteristic:** eSewa is a **client-side redirect** gateway. Your Django server never initiates a direct HTTP request to eSewa during payment initiation — it only builds the payload. The browser does the rest.

---

## 2. The Complete Payment Lifecycle

Here is exactly what happens from the moment a user clicks "Pay with eSewa" to when your system marks the order as paid:

```
┌─────────┐         ┌──────────┐         ┌──────────┐         ┌──────────┐
│ Frontend │         │  Django  │         │  eSewa   │         │  Django  │
│  (React) │         │ Backend  │         │ Servers  │         │ Callback │
└────┬─────┘         └────┬─────┘         └────┬─────┘         └────┬─────┘
     │                    │                    │                    │
     │  1. POST /orders/  │                    │                    │
     │  {payment: "eSewa"}│                    │                    │
     │───────────────────>│                    │                    │
     │                    │                    │                    │
     │                    │ 2. Create Order    │                    │
     │                    │    Generate HMAC   │                    │
     │                    │    Build payload   │                    │
     │                    │                    │                    │
     │  3. Return payload │                    │                    │
     │<───────────────────│                    │                    │
     │                    │                    │                    │
     │  4. Auto-submit    │                    │                    │
     │     hidden form ──────────────────────>│                    │
     │                    │                    │                    │
     │                    │                    │ 5. User logs in    │
     │                    │                    │    and pays        │
     │                    │                    │                    │
     │  6. Redirect to success_url with       │                    │
     │     Base64-encoded receipt              │                    │
     │<───────────────────────────────────────│                    │
     │                    │                    │                    │
     │  7. Forward receipt to callback         │                    │
     │─────────────────────────────────────────────────────────────>│
     │                    │                    │                    │
     │                    │                    │   8. Decode Base64 │
     │                    │                    │      Verify status │
     │                    │                    │      Update order  │
     │                    │                    │                    │
     │  9. "Payment successful"                                    │
     │<────────────────────────────────────────────────────────────│
```

### Step-by-step breakdown:

1. **User places order** — Frontend sends `POST /api/orders/` with `payment_method: "eSewa"`.
2. **Backend creates the order** — Django creates the Order record and calculates the HMAC-SHA256 signature using the order amount, order code, and merchant credentials.
3. **Backend returns payload** — Instead of a payment URL, it returns a structured object containing all the form fields eSewa needs.
4. **Frontend auto-submits a hidden form** — The React/Next.js frontend dynamically creates an HTML `<form>` with all those fields, sets its `action` to eSewa's payment URL, and submits it. The user's browser navigates to eSewa.
5. **User pays on eSewa** — The user sees eSewa's payment page, enters their PIN, and completes payment.
6. **eSewa redirects back** — After payment, eSewa redirects the browser to your `success_url` with a `?data=...` query parameter containing a Base64-encoded JSON receipt.
7. **Frontend forwards to callback** — The `success_url` points to your Django callback endpoint.
8. **Backend verifies** — Django decodes the Base64 data, checks that `status == "COMPLETE"`, and marks the order as paid.
9. **User sees confirmation** — The frontend displays a success message.

---

## 3. Understanding HMAC-SHA256 Signatures

The signature is an **anti-tampering mechanism**. Without it, a malicious user could change the amount from ₨1000 to ₨1 in their browser's developer tools before submitting the form. The signature prevents this.

### How it works:

```
Your data:    "total_amount=1000.00,transaction_uuid=ABC123,product_code=EPAYTEST"
Your secret:  "8gBm/:&EnhH.1/q"

                    ┌──────────────┐
  data + secret ──> │ HMAC-SHA256  │ ──> Binary hash ──> Base64 encode ──> "x7Fk9a..."
                    └──────────────┘
```

- **Only you and eSewa know the secret key.** So only you can produce a valid signature for a given amount/order.
- **If someone changes the amount,** the signature won't match and eSewa will reject the request.
- **The algorithm is HMAC-SHA256** — a standard, well-audited cryptographic hash. The output is Base64-encoded (not hex).

### Critical rules:

| Rule | Why |
|---|---|
| No spaces after commas | `total_amount=100,transaction_uuid=X` not `total_amount=100, transaction_uuid=X` |
| Fields must be in exact order | `total_amount`, then `transaction_uuid`, then `product_code` |
| Amount format must match exactly | `"1000.00"` in signature must match `"1000.00"` in the form field |

---

## 4. Prerequisites

Before you begin, make sure you have:

- **Python 3.10+** and **Django 4.2+** (or 5.x/6.x)
- **Django REST Framework** installed
- **An eSewa Merchant Account** — For testing, eSewa provides public sandbox credentials (no registration needed)

Install the required package (you likely already have it):

```bash
pip install djangorestframework
```

No additional SDK or library is needed — eSewa integration uses standard Python `hmac` and `base64` modules.

---

## Step 1 — Django Settings Configuration

Add the eSewa configuration block to your `settings.py`:

```python
# settings.py

ESEWA_SETTINGS = {
    # Sandbox credentials (public — anyone can use these for testing)
    "MERCHANT_ID": "EPAYTEST",
    "SECRET_KEY": "8gBm/:&EnhH.1/q",
    
    # eSewa epay v2 sandbox endpoint
    "INITIATE_URL": "https://rc-epay.esewa.com.np/api/epay/main/v2/form",
    
    # Where eSewa redirects after payment (your Django callback)
    "SUCCESS_URL": "http://localhost:8000/api/orders/esewa-verify/",
    
    # Where eSewa redirects on cancel/failure (your frontend)
    "FAILURE_URL": "http://localhost:3000/payment-failed",
}
```

**For production**, you'll replace:
- `MERCHANT_ID` → Your real merchant code from eSewa's merchant portal
- `SECRET_KEY` → Your real secret key
- `INITIATE_URL` → `"https://epay.esewa.com.np/api/epay/main/v2/form"`
- `SUCCESS_URL` → Your production callback URL (must be HTTPS)
- `FAILURE_URL` → Your production frontend failure page (must be HTTPS)

---

## Step 2 — The Order Model

Your Order model needs these fields at minimum:

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

    # Unique order identifier — this becomes eSewa's `transaction_uuid`
    order_code = models.CharField(max_length=10, unique=True)
    
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    
    payment_method = models.CharField(max_length=50, choices=PAYMENT_METHODS)
    payment_status = models.CharField(max_length=20, choices=PAYMENT_STATUS, default='Pending')
    
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    
    # eSewa-specific: store the transaction reference returned after payment
    transaction_id = models.CharField(max_length=100, null=True, blank=True)
    
    # Store the raw response from eSewa for debugging/auditing
    payment_response = models.JSONField(null=True, blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)
```

### Why `order_code` matters:

eSewa calls this the `transaction_uuid`. It must be:
- **Unique per transaction** — eSewa will reject duplicate UUIDs
- **Short and alphanumeric** — Ideally 6-10 characters
- **Stored in your database** — So you can match the callback to the order

---

## Step 3 — Signature Generation Utility

This is the most critical piece. Get one character wrong and eSewa will silently reject the payment.

```python
# orders/utils.py

import hmac
import hashlib
import base64

def generate_esewa_signature(total_amount, transaction_uuid, product_code, secret_key):
    """
    Generate HMAC-SHA256 signature for eSewa epay v2.
    
    CRITICAL FORMAT RULES:
    - Fields are separated by commas with NO spaces
    - Fields must be in this exact order: total_amount, transaction_uuid, product_code
    - Each field is in key=value format
    - The output is Base64-encoded (not hex)
    
    Args:
        total_amount:     String like "1000.00" — must match the form field exactly
        transaction_uuid: Your order_code (e.g., "ABC123")
        product_code:     Your MERCHANT_ID (e.g., "EPAYTEST")
        secret_key:       Your eSewa secret key
    
    Returns:
        Base64-encoded HMAC-SHA256 signature string
    """
    # Build the data string — DO NOT add spaces after commas
    data = f"total_amount={total_amount},transaction_uuid={transaction_uuid},product_code={product_code}"
    
    # Encode both to bytes
    secret_key_bytes = secret_key.encode('utf-8')
    data_bytes = data.encode('utf-8')
    
    # Create the HMAC-SHA256 hash
    hash_obj = hmac.new(secret_key_bytes, data_bytes, hashlib.sha256).digest()
    
    # Return as Base64 string
    return base64.b64encode(hash_obj).decode('utf-8')
```

### Testing the signature in isolation:

```python
# Quick sanity check — run in Django shell
from orders.utils import generate_esewa_signature

sig = generate_esewa_signature(
    total_amount="100.00",
    transaction_uuid="TEST001",
    product_code="EPAYTEST",
    secret_key="8gBm/:&EnhH.1/q"
)
print(sig)  # Should output a Base64 string like "Ly0xP3B2..."
print(len(sig))  # Should be ~44 characters
```

---

## Step 4 — Payment Strategy (Building the Payload)

This class builds the complete form payload that the frontend will submit to eSewa:

```python
# orders/payment_strategies.py

from django.conf import settings
from .utils import generate_esewa_signature

class EsewaStrategy:
    """
    Builds the eSewa payment form payload.
    
    eSewa uses a form-submission model: we build all the hidden form fields
    here, and the frontend creates an actual <form> element and submits it.
    """
    
    def get_payment_payload(self, order):
        conf = settings.ESEWA_SETTINGS
        
        # Format amount to exactly 2 decimal places
        # "100" and "100.00" produce DIFFERENT signatures — always use .2f
        amount_str = "{:.2f}".format(order.total_amount)
        
        # Generate the tamper-proof signature
        signature = generate_esewa_signature(
            total_amount=amount_str,
            transaction_uuid=order.order_code,
            product_code=conf["MERCHANT_ID"],
            secret_key=conf["SECRET_KEY"]
        )

        return {
            "payment_method": "eSewa",
            "esewa_payload": {
                # Amount fields — all strings
                "amount": amount_str,
                "tax_amount": "0",
                "total_amount": amount_str,
                "product_service_charge": "0",
                "product_delivery_charge": "0",
                
                # Transaction identification
                "transaction_uuid": order.order_code,
                "product_code": conf["MERCHANT_ID"],
                
                # Callback URLs
                "success_url": conf["SUCCESS_URL"],
                "failure_url": conf["FAILURE_URL"],
                
                # Signature fields
                "signed_field_names": "total_amount,transaction_uuid,product_code",
                "signature": signature,
                
                # The URL the form should be submitted to
                "esewa_url": conf["INITIATE_URL"]
            }
        }
```

### Field-by-field explanation:

| Field | Purpose | Example Value |
|---|---|---|
| `amount` | Base price (before tax/charges) | `"1500.00"` |
| `tax_amount` | Tax amount (can be 0) | `"0"` |
| `total_amount` | Final amount to charge. **This goes into the signature.** | `"1500.00"` |
| `product_service_charge` | Service fee (can be 0) | `"0"` |
| `product_delivery_charge` | Delivery fee (can be 0) | `"0"` |
| `transaction_uuid` | Your unique order code. **This goes into the signature.** | `"A1B2C3"` |
| `product_code` | Your eSewa merchant ID. **This goes into the signature.** | `"EPAYTEST"` |
| `success_url` | Where eSewa redirects after successful payment | `"https://yoursite.com/api/orders/esewa-verify/"` |
| `failure_url` | Where eSewa redirects if user cancels | `"https://yoursite.com/payment-failed"` |
| `signed_field_names` | Tells eSewa which fields are in the signature | `"total_amount,transaction_uuid,product_code"` |
| `signature` | The HMAC-SHA256 hash | `"x7Fk9a..."` |
| `esewa_url` | The eSewa endpoint to submit the form to | `"https://rc-epay.esewa.com.np/..."` |

> **Important:** `amount + tax_amount + product_service_charge + product_delivery_charge` must equal `total_amount`. If they don't add up, eSewa may reject the request silently.

---

## Step 5 — The Order Creation View

When a user places an order with "eSewa" as the payment method, return the payload instead of a simple confirmation:

```python
# orders/views.py

from rest_framework import viewsets, status
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from .models import Order
from .payment_strategies import EsewaStrategy

class OrderViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]

    def create(self, request, *args, **kwargs):
        # ... your order creation logic here ...
        # After creating the order:
        
        order = ...  # Your newly created Order instance
        
        if order.payment_method == 'eSewa':
            strategy = EsewaStrategy()
            payment_data = strategy.get_payment_payload(order)
            
            return Response({
                "message": "Order initiated. Proceed to payment.",
                **payment_data
            }, status=status.HTTP_201_CREATED)
        
        # Handle other payment methods...
```

### What the API response looks like:

```json
{
    "message": "Order initiated. Proceed to payment.",
    "payment_method": "eSewa",
    "esewa_payload": {
        "amount": "1500.00",
        "tax_amount": "0",
        "total_amount": "1500.00",
        "transaction_uuid": "A1B2C3",
        "product_code": "EPAYTEST",
        "product_service_charge": "0",
        "product_delivery_charge": "0",
        "success_url": "http://localhost:8000/api/orders/esewa-verify/",
        "failure_url": "http://localhost:3000/payment-failed",
        "signed_field_names": "total_amount,transaction_uuid,product_code",
        "signature": "dGhpcyBpcyBhIHNhbXBsZSBzaWduYXR1cmU=",
        "esewa_url": "https://rc-epay.esewa.com.np/api/epay/main/v2/form"
    }
}
```

---

## Step 6 — The Verification Callback View

After the user pays, eSewa redirects them to your `success_url` with a Base64-encoded JSON string in the `data` query parameter. You need to decode it and verify the payment.

```python
# orders/views.py

from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from django.shortcuts import get_object_or_404
import base64
import json
from .models import Order

class PaymentCallbackViewSet(viewsets.ViewSet):
    """
    Handles the redirect callback from eSewa after payment.
    
    eSewa sends: GET /api/orders/esewa-verify/?data=eyJzdGF0dXMi...
    The `data` param is a Base64-encoded JSON string.
    """
    
    @action(detail=False, methods=['get'], url_path='esewa-verify')
    def esewa_verify(self, request):
        # 1. Get the encoded data from the query parameters
        encoded_data = request.query_params.get('data')
        if not encoded_data:
            return Response({"error": "No data from eSewa"}, status=400)

        try:
            # 2. Decode the Base64 string into JSON
            decoded_bytes = base64.b64decode(encoded_data)
            decoded_data = json.loads(decoded_bytes.decode('utf-8'))
            
            """
            decoded_data looks like:
            {
                "transaction_code": "0007XYZ",     <- eSewa's reference ID
                "status": "COMPLETE",               <- or "INCOMPLETE", "PENDING"
                "total_amount": "1,500.0",          
                "transaction_uuid": "A1B2C3",       <- Your order_code
                "product_code": "EPAYTEST",
                "signed_field_names": "...",
                "signature": "..."
            }
            """
            
            # 3. Check the payment status
            status_received = decoded_data.get('status')
            order_code = decoded_data.get('transaction_uuid')
            esewa_ref = decoded_data.get('transaction_code')
            
            # 4. If payment was completed, update the order
            if status_received == "COMPLETE":
                order = get_object_or_404(Order, order_code=order_code)
                order.payment_status = "Completed"
                order.transaction_id = esewa_ref
                order.payment_response = decoded_data  # Store raw data for auditing
                order.save()
                
                return Response({
                    "message": "Payment successful",
                    "order": order.order_code
                })
            
            return Response({"error": "Payment was not completed"}, status=400)
            
        except Exception as e:
            return Response({"error": str(e)}, status=400)
```

### What the decoded eSewa response contains:

| Field | Description |
|---|---|
| `transaction_code` | eSewa's own transaction reference (e.g., `"0007XYZ"`) |
| `status` | `"COMPLETE"`, `"INCOMPLETE"`, or `"PENDING"` |
| `total_amount` | The amount that was charged (may include commas: `"1,500.0"`) |
| `transaction_uuid` | Your `order_code` — use this to find the order |
| `product_code` | Your merchant ID — should match your settings |
| `signed_field_names` | Which fields were signed |
| `signature` | eSewa's signature — you can verify this for extra security |

---

## Step 7 — URL Configuration

Wire up the callback endpoint:

```python
# orders/urls.py

from django.urls import path, include
from rest_framework.routers import SimpleRouter
from .views import OrderViewSet, PaymentCallbackViewSet

router = SimpleRouter()
router.register(r'', OrderViewSet, basename='order')

urlpatterns = [
    # eSewa payment callback — must match your SUCCESS_URL path
    path('esewa-verify/', PaymentCallbackViewSet.as_view({'get': 'esewa_verify'}), name='esewa-verify'),
    path('', include(router.urls)),
]
```

And in your main `urls.py`:

```python
# project/urls.py

urlpatterns = [
    path('api/orders/', include('orders.urls')),
    # ... other paths
]
```

This gives you: `GET /api/orders/esewa-verify/` — which must match the `SUCCESS_URL` in your settings.

---

## Step 8 — Frontend Integration

The frontend receives the `esewa_payload` and needs to submit it as an HTML form. Here's how in React:

```jsx
// React component
const handleEsewaPayment = (esewaPayload) => {
    // Create a hidden form and auto-submit it
    const form = document.createElement('form');
    form.method = 'POST';
    form.action = esewaPayload.esewa_url; // eSewa's payment page URL
    
    // Add all payload fields as hidden inputs
    Object.entries(esewaPayload).forEach(([key, value]) => {
        if (key === 'esewa_url') return; // Skip — this is the form action, not a field
        
        const input = document.createElement('input');
        input.type = 'hidden';
        input.name = key;
        input.value = value;
        form.appendChild(input);
    });
    
    // Append to body and submit — this navigates the browser to eSewa
    document.body.appendChild(form);
    form.submit();
};

// Usage in your order placement handler:
const placeOrder = async () => {
    const response = await fetch('/api/orders/', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${token}` },
        body: JSON.stringify({ address_id: 5, payment_method: 'eSewa' })
    });
    
    const data = await response.json();
    
    if (data.payment_method === 'eSewa') {
        handleEsewaPayment(data.esewa_payload);
    }
};
```

---

## Testing with Sandbox Credentials

eSewa provides **public sandbox credentials** that anyone can use:

| Setting | Sandbox Value |
|---|---|
| Merchant ID | `EPAYTEST` |
| Secret Key | `8gBm/:&EnhH.1/q` |
| Payment URL | `https://rc-epay.esewa.com.np/api/epay/main/v2/form` |

### Sandbox test accounts:

eSewa provides test wallet accounts at [https://developer.esewa.com.np/pages/epay](https://developer.esewa.com.np/pages/epay). You can use their test credentials to complete sandbox payments.

### Quick test — HTML form:

Save this as an HTML file and open it in your browser:

```html
<!DOCTYPE html>
<html>
<body>
    <h2>eSewa Sandbox Test</h2>
    <form action="https://rc-epay.esewa.com.np/api/epay/main/v2/form" method="POST">
        <input type="hidden" name="amount" value="100.00">
        <input type="hidden" name="tax_amount" value="0">
        <input type="hidden" name="total_amount" value="100.00">
        <input type="hidden" name="transaction_uuid" value="TEST_001">
        <input type="hidden" name="product_code" value="EPAYTEST">
        <input type="hidden" name="product_service_charge" value="0">
        <input type="hidden" name="product_delivery_charge" value="0">
        <input type="hidden" name="success_url" value="http://localhost:8000/api/orders/esewa-verify/">
        <input type="hidden" name="failure_url" value="http://localhost:3000/payment-failed">
        <input type="hidden" name="signed_field_names" value="total_amount,transaction_uuid,product_code">
        <input type="hidden" name="signature" value="PUT_YOUR_GENERATED_SIGNATURE_HERE">
        <button type="submit">Pay with eSewa (Test)</button>
    </form>
</body>
</html>
```

Generate the signature using `python manage.py shell`:

```python
from orders.utils import generate_esewa_signature
sig = generate_esewa_signature("100.00", "TEST_001", "EPAYTEST", "8gBm/:&EnhH.1/q")
print(sig)
```

Paste the output into the HTML file's signature field.

---

## Production Checklist

Before going live, ensure:

- [ ] Replace sandbox credentials with your real eSewa merchant credentials
- [ ] Change `INITIATE_URL` to `https://epay.esewa.com.np/api/epay/main/v2/form`
- [ ] Update `SUCCESS_URL` and `FAILURE_URL` to use HTTPS production domains
- [ ] Move `SECRET_KEY` to environment variables (never hardcode in production)
- [ ] Add signature verification on the callback response for extra security
- [ ] Add idempotency checks — ensure the same callback doesn't update an order twice
- [ ] Handle edge cases: what if the user closes the browser during payment?
- [ ] Log all payment callbacks for auditing

---

## Common Pitfalls & Debugging

### 1. "Invalid signature" or payment page shows error

**Cause:** The signature string doesn't match exactly.

**Debug steps:**
```python
# Print the exact string being hashed
data = f"total_amount=100.00,transaction_uuid=TEST001,product_code=EPAYTEST"
print(repr(data))  # Look for hidden spaces, newlines, encoding issues
```

Common mistakes:
- Adding spaces after commas: `total_amount=100, transaction_uuid=...` ✘
- Wrong field order: `product_code` before `total_amount` ✘
- Amount mismatch: signature uses `"100"` but form sends `"100.00"` ✘

### 2. Callback never fires

**Cause:** `SUCCESS_URL` is wrong or unreachable.

- Make sure the URL is accessible from the browser (not just your server)
- For local development, eSewa redirects the *browser*, so `localhost` works
- For sandbox, `http://` is fine; for production, must be `https://`

### 3. Decoded callback data has unexpected format

**Cause:** eSewa's response format may vary between versions.

Always wrap your decode logic in try/except and log the raw `encoded_data` string before attempting to decode, so you can debug if the format changes.

### 4. Amount field formatting

**Cause:** Django's `DecimalField` may format differently than expected.

Always explicitly format: `"{:.2f}".format(float(order.total_amount))` — never pass the raw Decimal object.

---

*Last updated: April 2026. Based on eSewa epay API v2.*
