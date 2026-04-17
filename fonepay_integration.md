# Fonepay Payment Gateway Integration — Django REST Framework

> A complete, production-ready guide to integrating Fonepay's Merchant API (v2.0) into a Django/DRF backend for Nepali e-commerce applications.
### By Prashant Maharjan
---

## Table of Contents

1. [How Fonepay Actually Works](#1-how-fonepay-actually-works)
2. [The Complete Payment Lifecycle](#2-the-complete-payment-lifecycle)
3. [Understanding HMAC-SHA512 Signatures](#3-understanding-hmac-sha512-signatures)
4. [Fonepay vs eSewa vs Khalti — Comparison](#4-fonepay-vs-esewa-vs-khalti--comparison)
5. [Prerequisites](#5-prerequisites)
6. [Step 1 — Django Settings Configuration](#step-1--django-settings-configuration)
7. [Step 2 — The Order Model](#step-2--the-order-model)
8. [Step 3 — Signature Generation Utility](#step-3--signature-generation-utility)
9. [Step 4 — Payment Strategy (Building the Payment URL)](#step-4--payment-strategy-building-the-payment-url)
10. [Step 5 — The Order Creation View](#step-5--the-order-creation-view)
11. [Step 6 — The Verification Callback View](#step-6--the-verification-callback-view)
12. [Step 7 — URL Configuration](#step-7--url-configuration)
13. [Step 8 — Frontend Integration](#step-8--frontend-integration)
14. [Testing & Sandbox Configuration](#testing--sandbox-configuration)
15. [Production Checklist](#production-checklist)
16. [Common Pitfalls & Debugging](#common-pitfalls--debugging)

---

## 1. How Fonepay Actually Works

Fonepay is Nepal's interbank payment network — the rails that connect all banks. Unlike eSewa or Khalti (which are wallets), Fonepay lets customers pay **directly from their bank accounts** via mobile banking apps. When a customer selects Fonepay, they see a list of all banks, pick theirs, log into their mobile banking, and authorize the payment.

### The Mental Model

Think of Fonepay like a **URL-based redirect gateway with cryptographic signing**. Instead of submitting a form (like eSewa) or calling an API (like Khalti), you build a specially-crafted URL with all payment parameters embedded as query strings and a tamper-proof hash (called `DV` — Data Validation). The user's browser navigates to this URL, Fonepay presents the bank selection page, and after payment, Fonepay redirects back to your server with the result.

The key differentiator: **Fonepay uses HMAC-SHA512** (stronger than eSewa's SHA256) and the hash is in **uppercase hexadecimal** (not Base64 like eSewa).

### How Fonepay fits in the Nepal payments ecosystem:

```
                    ┌──────────────────────────────────────┐
                    │         FONEPAY NETWORK               │
                    │                                      │
     ┌──────────┐   │   ┌─────────┐   ┌─────────────────┐ │
     │ Your     │───│──>│ Fonepay │──>│ Customer's Bank  │ │
     │ Website  │   │   │ Gateway │   │ (NIC Asia, etc.) │ │
     └──────────┘   │   └─────────┘   └─────────────────┘ │
                    │                                      │
                    │   Banks connected: Global IME, NIC   │
                    │   Asia, Laxmi, Machhapuchchhre, etc.  │
                    └──────────────────────────────────────┘
```

---

## 2. The Complete Payment Lifecycle

```
┌─────────┐         ┌──────────┐         ┌──────────┐         ┌──────────┐
│ Frontend │         │  Django  │         │ Fonepay  │         │  Django  │
│  (React) │         │ Backend  │         │ Servers  │         │ Callback │
└────┬─────┘         └────┬─────┘         └────┬─────┘         └────┬─────┘
     │                    │                    │                    │
     │  1. POST /orders/  │                    │                    │
     │  {payment:         │                    │                    │
     │   "Fonepay"}       │                    │                    │
     │───────────────────>│                    │                    │
     │                    │                    │                    │
     │                    │ 2. Create Order    │                    │
     │                    │    Build params    │                    │
     │                    │    Generate DV     │                    │
     │                    │    (HMAC-SHA512)   │                    │
     │                    │    Build full URL  │                    │
     │                    │                    │                    │
     │  3. Return         │                    │                    │
     │     {full_url}     │                    │                    │
     │<───────────────────│                    │                    │
     │                    │                    │                    │
     │  4. window.location = full_url          │                    │
     │────────────────────────────────────────>│                    │
     │                    │                    │                    │
     │                    │                    │ 5. User selects    │
     │                    │                    │    bank, logs in,  │
     │                    │                    │    and pays        │
     │                    │                    │                    │
     │  6. Redirect to RU (your callback)      │                    │
     │     with ?PRN=...&PS=true&RC=successful │                    │
     │     &UID=...&DV=...                     │                    │
     │<───────────────────────────────────────│                    │
     │                    │                    │                    │
     │  7. Browser hits callback               │                    │
     │─────────────────────────────────────────────────────────────>│
     │                    │                    │                    │
     │                    │                    │   8. Verify with   │
     │                    │                    │      Fonepay API   │
     │                    │                    │<────────────────────│
     │                    │                    │                    │
     │                    │                    │   9. Confirm       │
     │                    │                    │─────────────────────>│
     │                    │                    │                    │
     │                    │                    │  10. Update order  │
     │                    │                    │                    │
     │  11. "Payment successful"                                   │
     │<────────────────────────────────────────────────────────────│
```

### Step-by-step breakdown:

1. **User places order** — Frontend sends `POST /api/orders/` with `payment_method: "Fonepay"`.
2. **Backend builds the payment URL** — Django creates the Order, assembles all required parameters (PID, PRN, AMT, DT, etc.), generates the HMAC-SHA512 signature (DV), and constructs the full redirect URL.
3. **Backend returns the URL** — The response includes `full_url` — the complete Fonepay payment URL with all params.
4. **Frontend redirects the user** — `window.location.href = full_url` takes the user to Fonepay.
5. **User pays via their bank** — Fonepay shows a bank selection page. The user picks their bank, opens their mobile banking app, and authorizes the payment.
6. **Fonepay redirects back** — After payment, Fonepay redirects the browser to your `RU` (Return URL) with query parameters: `PRN`, `PS`, `RC`, `UID`, `DV`, etc.
7. **Callback receives the request** — Your Django view processes the incoming GET request.
8. **Server-side verification** — Your backend calls Fonepay's verification endpoint to confirm the payment.
9. **Fonepay confirms** — Returns a success response.
10. **Order is updated** — Mark the order as paid.
11. **User sees confirmation**.

---

## 3. Understanding HMAC-SHA512 Signatures

Fonepay's DV (Data Validation) hash is the cornerstone of the integration. It prevents anyone from tampering with payment parameters.

### How the DV hash is generated:

```
Step 1: Concatenate fields in EXACT order, separated by commas:
        "PID,MD,PRN,AMT,CRN,DT,R1,R2,RU"

Step 2: HMAC-SHA512 hash with your secret key:
        HMAC-SHA512(secret_key, data_string)

Step 3: Convert to UPPERCASE hexadecimal:
        "BFC4A4658DD04EF032DD..."
```

### Visual example:

```
Your data:
    PID = "2222090019536084"
    MD  = "P"
    PRN = "A1B2C3"
    AMT = "1500.00"
    CRN = "NPR"
    DT  = "04/16/2026"
    R1  = "Order_A1B2C3"
    R2  = "N/A"
    RU  = "https://yoursite.com/api/orders/fonepay-verify/"

Concatenated:
    "2222090019536084,P,A1B2C3,1500.00,NPR,04/16/2026,Order_A1B2C3,N/A,https://yoursite.com/api/orders/fonepay-verify/"

                    ┌──────────────┐
  data + secret ──> │ HMAC-SHA512  │ ──> hex string ──> UPPERCASE ──> "BFC4A4658D..."
                    └──────────────┘
```

### Critical rules:

| Rule | Details |
|---|---|
| **Field order is sacred** | `PID,MD,PRN,AMT,CRN,DT,R1,R2,RU` — this exact order. |
| **No URL encoding in the hash** | The date `04/16/2026` goes into the hash as-is, not as `04%2F16%2F2026`. |
| **Hash output must be UPPERCASE** | `bfc4a4...` ✘ → `BFC4A4...` ✔ |
| **Same values in hash and URL** | If the hash uses `AMT=1500.00`, the URL param must also be `AMT=1500.00`. |
| **R2 can be "N/A"** | Fonepay docs say to use `"N/A"` if you don't have additional info. |
| **RU must be HTTPS in production** | Fonepay's production environment requires HTTPS return URLs. |

---

## 4. Fonepay vs eSewa vs Khalti — Comparison

| Aspect | Fonepay | eSewa | Khalti |
|---|---|---|---|
| **Type** | Interbank payment network | Digital wallet | Digital wallet |
| **Payment source** | Bank accounts | eSewa wallet | Khalti wallet |
| **Integration style** | URL redirect with query params | HTML form POST | Server-to-server REST API |
| **Signature algorithm** | HMAC-SHA512 (hex, uppercase) | HMAC-SHA256 (Base64) | API key in header |
| **Amount format** | String with 2 decimals (`"1500.00"`) | String with 2 decimals (`"1500.00"`) | Integer in paisa (`150000`) |
| **Server-to-server call at initiation?** | No | No | Yes |
| **Verification** | GET/POST to Fonepay verify endpoint | Decode Base64 callback data | POST to Khalti lookup API |
| **Credentials obtained from** | Your bank (partner bank) | eSewa merchant portal | Khalti merchant portal |

---

## 5. Prerequisites

Before you begin:

- **Python 3.10+** and **Django 4.2+** (or 5.x/6.x)
- **Django REST Framework** installed
- **`requests` library** installed (for server-side verification)
- **Fonepay Merchant Credentials** — PID and Secret Key obtained from your acquiring bank
- **HTTPS callback URL** — Fonepay production requires HTTPS. Use [ngrok](https://ngrok.com) for development.

```bash
pip install djangorestframework requests
```

> **Important:** Unlike eSewa or Khalti, Fonepay does **not** have public test credentials. You must register with a Fonepay partner bank to obtain your PID and Secret Key. This is typically done through the digital banking department of your business bank (e.g., Global IME, NIC Asia, Laxmi Bank).

---

## Step 1 — Django Settings Configuration

```python
# settings.py

FONEPAY_SETTINGS = {
    # Production URLs
    "INITIATE_URL": "https://clientapi.fonepay.com/api/merchantRequest",
    "VERIFY_URL": "https://clientapi.fonepay.com/api/merchantRequest/verificationMerchant",
    
    # Sandbox URLs (if your merchant account is on sandbox)
    # "INITIATE_URL": "https://dev-clientapi.fonepay.com/api/merchantRequest",
    # "VERIFY_URL": "https://dev-clientapi.fonepay.com/api/merchantRequest/verificationMerchant",
    
    # Your credentials — obtained from your partner bank
    "MERCHANT_ID": "your_pid_here",       # Also called PID
    "SECRET_KEY": "your_secret_key_here", # Used for HMAC-SHA512 signing
    
    # Where Fonepay redirects after payment (must be HTTPS in production)
    "SUCCESS_URL": "https://yourdomain.com/api/orders/fonepay-verify/",
}
```

### Environment-specific configuration:

For development with ngrok:
```python
# Development — using ngrok tunnel
FONEPAY_SETTINGS = {
    "INITIATE_URL": "https://clientapi.fonepay.com/api/merchantRequest",
    "VERIFY_URL": "https://clientapi.fonepay.com/api/merchantRequest/verificationMerchant",
    "MERCHANT_ID": "your_pid_here",
    "SECRET_KEY": "your_secret_key_here",
    "SUCCESS_URL": "https://your-tunnel.ngrok-free.dev/api/orders/fonepay-verify/",
}
```

> **Why ngrok?** Fonepay redirects the user's browser to your `SUCCESS_URL` after payment. If you're running locally, `http://localhost:8000/...` won't work because Fonepay can't reach your local machine. Use `ngrok http 8000` to create a public URL that tunnels to your local server.

---

## Step 2 — The Order Model

The Order model for Fonepay is the same base model used for other gateways:

```python
# orders/models.py

from django.db import models
from django.conf import settings

class Order(models.Model):
    PAYMENT_METHODS = (
        ('eSewa', 'eSewa'),
        ('Khalti', 'Khalti'),
        ('Fonepay', 'Fonepay'),
        ('COD', 'Cash on Delivery'),
    )

    PAYMENT_STATUS = (
        ('Pending', 'Pending'),
        ('Completed', 'Completed'),
        ('Failed', 'Failed'),
    )

    # Unique order identifier — this becomes Fonepay's PRN
    order_code = models.CharField(max_length=10, unique=True)
    
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE)
    
    payment_method = models.CharField(max_length=50, choices=PAYMENT_METHODS)
    payment_status = models.CharField(max_length=20, choices=PAYMENT_STATUS, default='Pending')
    
    total_amount = models.DecimalField(max_digits=10, decimal_places=2)
    
    # Fonepay-specific: store the UID (Fonepay Trace ID) after payment
    transaction_id = models.CharField(max_length=100, null=True, blank=True)
    
    created_at = models.DateTimeField(auto_now_add=True)
```

### Fonepay-specific field mapping:

| Fonepay Parameter | Your Model Field | Description |
|---|---|---|
| `PRN` | `order_code` | Product Reference Number — your unique transaction ID |
| `AMT` | `total_amount` | Payment amount (formatted to 2 decimals) |
| `DT` | `created_at` | Transaction date (formatted as `MM/DD/YYYY`) |
| `UID` (response) | `transaction_id` | Fonepay's Trace ID (stored after verification) |

---

## Step 3 — Signature Generation Utility

The signature function is the heart of Fonepay integration. It must produce an HMAC-SHA512 hash in uppercase hex:

```python
# orders/utils.py

import hmac
import hashlib

def generate_fonepay_signature(prn, amt, pid, ru, secret_key, r1, r2, dt):
    """
    Generate HMAC-SHA512 signature (DV) for Fonepay Merchant API v2.0.
    
    CRITICAL: The field order in the data string is MANDATORY and must be:
    PID, MD, PRN, AMT, CRN, DT, R1, R2, RU
    
    The values must NOT be URL-encoded when computing the hash.
    The output must be UPPERCASE hexadecimal.
    
    Args:
        prn:        Product Reference Number (your order_code)
        amt:        Amount as string with 2 decimals (e.g., "1500.00")
        pid:        Your Merchant ID (PID)
        ru:         Return URL (your callback endpoint)
        secret_key: Your Fonepay shared secret key
        r1:         Payment details / description
        r2:         Additional info (use "N/A" if not applicable)
        dt:         Date in MM/DD/YYYY format
    
    Returns:
        Uppercase hexadecimal HMAC-SHA512 hash (128 characters)
    """
    # MD is always 'P' for payment requests
    md = 'P'
    
    # Build the data string — NO URL encoding, separated by commas
    # Order: PID, MD, PRN, AMT, CRN, DT, R1, R2, RU
    data = f"{pid},{md},{prn},{amt},NPR,{dt},{r1},{r2},{ru}"
    
    # Generate HMAC-SHA512
    hash_result = hmac.new(
        secret_key.encode('utf-8'),
        data.encode('utf-8'),
        hashlib.sha512
    ).hexdigest()
    
    # Fonepay requires UPPERCASE hex
    return hash_result.upper()
```

### Testing the signature in isolation:

```python
# Run in Django shell: python manage.py shell
from orders.utils import generate_fonepay_signature

dv = generate_fonepay_signature(
    prn="TEST001",
    amt="100.00",
    pid="2222090019536084",
    ru="https://example.com/api/orders/fonepay-verify/",
    secret_key="7dcfe29643d64cce873157a940ad05d1",
    r1="Order_TEST001",
    r2="N/A",
    dt="04/16/2026"
)
print(dv)        # Should be 128 character uppercase hex string
print(len(dv))   # Should print 128
```

### Verifying correctness manually:

You can cross-check with Python's built-in tools:

```python
import hmac, hashlib

data = "2222090019536084,P,TEST001,100.00,NPR,04/16/2026,Order_TEST001,N/A,https://example.com/api/orders/fonepay-verify/"
key = "7dcfe29643d64cce873157a940ad05d1"

result = hmac.new(key.encode(), data.encode(), hashlib.sha512).hexdigest().upper()
print(result)
print(result == dv)  # Should print True
```

---

## Step 4 — Payment Strategy (Building the Payment URL)

Fonepay uses a URL-redirect model. Your backend assembles all parameters, signs them, and constructs a full URL. The frontend simply redirects to that URL:

```python
# orders/payment_strategies.py

from django.conf import settings
from .utils import generate_fonepay_signature
import urllib.parse
import logging

logger = logging.getLogger(__name__)

class FonepayStrategy:
    """
    Builds the Fonepay payment redirect URL.
    
    Fonepay is URL-based: all parameters are passed as query strings
    in a GET request. The DV (Data Validation) hash ensures that
    the parameters haven't been tampered with.
    """
    
    def get_payment_payload(self, order):
        conf = settings.FONEPAY_SETTINGS
        
        # Format amount to exactly 2 decimal places
        # CRITICAL: Cast to float first to avoid Django Decimal formatting quirks
        amt = "{:.2f}".format(float(order.total_amount))
        
        # PRN = Product Reference Number = your unique order code
        prn = order.order_code
        
        # Date in MM/DD/YYYY format (Fonepay requirement)
        dt = order.created_at.strftime("%m/%d/%Y")
        
        # R1 = Payment description, R2 = Additional info
        r1 = f"Order_{prn}"
        r2 = "N/A"  # Fonepay docs: use "N/A" when the field is not applicable
        
        # MD = Mode. Always "P" for payment requests
        md = "P"

        # Generate the HMAC-SHA512 signature
        dv = generate_fonepay_signature(
            prn=prn, amt=amt, pid=conf["MERCHANT_ID"],
            ru=conf["SUCCESS_URL"], secret_key=conf["SECRET_KEY"],
            r1=r1, r2=r2, dt=dt
        )

        # Log for debugging (use DEBUG level, not print statements)
        logger.debug(
            "Fonepay signature input: %r",
            f"{conf['MERCHANT_ID']},{md},{prn},{amt},NPR,{dt},{r1},{r2},{conf['SUCCESS_URL']}"
        )

        # Build the query parameters — order matters for readability but
        # not for functionality (the DV hash is what matters)
        params = {
            "PID": conf["MERCHANT_ID"],  # Your Merchant ID
            "MD": md,                     # Payment Mode ("P")
            "PRN": prn,                   # Your order code
            "AMT": amt,                   # Amount (e.g., "1500.00")
            "CRN": "NPR",                # Currency
            "DT": dt,                     # Date (MM/DD/YYYY)
            "R1": r1,                     # Description
            "R2": r2,                     # Additional info
            "RU": conf["SUCCESS_URL"],    # Return URL (callback)
            "DV": dv,                     # The HMAC-SHA512 signature
        }

        # urllib.parse.urlencode handles URL-encoding of special chars
        # e.g., "/" in the date becomes "%2F", ":" in the URL becomes "%3A"
        query_string = urllib.parse.urlencode(params)
        full_url = f"{conf['INITIATE_URL']}?{query_string}"

        return {
            "payment_method": "Fonepay",
            "full_url": full_url,          # The user should be redirected here
            "fonepay_payload": params,      # Raw params for debugging
        }
```

### Field-by-field explanation:

| Field | Parameter | Example | Notes |
|---|---|---|---|
| Merchant ID | `PID` | `"2222090019536084"` | Assigned by your bank |
| Payment Mode | `MD` | `"P"` | Always "P" for payments |
| Order Reference | `PRN` | `"A1B2C3"` | Must be unique per transaction |
| Amount | `AMT` | `"1500.00"` | Two decimal places, as a string |
| Currency | `CRN` | `"NPR"` | Nepalese Rupees |
| Date | `DT` | `"04/16/2026"` | MM/DD/YYYY format |
| Description | `R1` | `"Order_A1B2C3"` | Payment details |
| Additional Info | `R2` | `"N/A"` | Use "N/A" if empty |
| Return URL | `RU` | `"https://..."` | Your callback endpoint (HTTPS) |
| Signature | `DV` | `"BFC4A4658D..."` | 128-char uppercase hex |

### What the generated URL looks like:

```
https://clientapi.fonepay.com/api/merchantRequest
    ?PID=2222090019536084
    &MD=P
    &PRN=A1B2C3
    &AMT=1500.00
    &CRN=NPR
    &DT=04%2F16%2F2026
    &R1=Order_A1B2C3
    &R2=N%2FA
    &RU=https%3A%2F%2Fyoursite.com%2Fapi%2Forders%2Ffonepay-verify%2F
    &DV=BFC4A4658DD04EF032DD255386B39C3697E3E6A226BF1E9A5CB80AAA28313E31...
```

> Note: The `DT` value `04/16/2026` becomes `04%2F16%2F2026` in the URL due to URL encoding. This is correct — the URL encoding is applied by `urllib.parse.urlencode()`, but the **signature was calculated on the raw, non-encoded value** `04/16/2026`.

---

## Step 5 — The Order Creation View

```python
# orders/views.py

from rest_framework import viewsets, status
from rest_framework.response import Response
from .models import Order
from .payment_strategies import FonepayStrategy

class OrderViewSet(viewsets.ModelViewSet):
    
    def create(self, request, *args, **kwargs):
        # ... your order creation logic ...
        order = ...  # Your newly created Order instance
        
        if order.payment_method == 'Fonepay':
            strategy = FonepayStrategy()
            payment_data = strategy.get_payment_payload(order)
            
            return Response({
                "message": "Order initiated. Proceed to payment.",
                **payment_data
            }, status=status.HTTP_201_CREATED)
```

### What the API response looks like:

```json
{
    "message": "Order initiated. Proceed to payment.",
    "payment_method": "Fonepay",
    "full_url": "https://clientapi.fonepay.com/api/merchantRequest?PID=2222090019536084&MD=P&PRN=A1B2C3&AMT=1500.00&CRN=NPR&DT=04%2F16%2F2026&R1=Order_A1B2C3&R2=N%2FA&RU=https%3A%2F%2Fyoursite.com%2Fapi%2Forders%2Ffonepay-verify%2F&DV=BFC4A4658DD04EF0...",
    "fonepay_payload": {
        "PID": "2222090019536084",
        "MD": "P",
        "PRN": "A1B2C3",
        "AMT": "1500.00",
        "CRN": "NPR",
        "DT": "04/16/2026",
        "R1": "Order_A1B2C3",
        "R2": "N/A",
        "RU": "https://yoursite.com/api/orders/fonepay-verify/",
        "DV": "BFC4A4658DD04EF032DD255386B39C3697E3E6A226BF1E9A5CB80AAA28313E31..."
    }
}
```

---

## Step 6 — The Verification Callback View

After the user completes (or cancels) the payment, Fonepay redirects their browser to your `RU` (Return URL) with the following query parameters:

| Parameter | Description | Example |
|---|---|---|
| `PRN` | Your order code | `"A1B2C3"` |
| `PID` | Your merchant ID | `"2222090019536084"` |
| `PS` | Payment Status (`"true"` or `"false"`) | `"true"` |
| `RC` | Response Code | `"successful"` |
| `DV` | Fonepay's DV hash of the response | `"ABC123..."` |
| `UID` | Fonepay Trace ID (their transaction reference) | `"1234567890"` |
| `P_AMT` | Actual amount paid | `"1500.00"` |
| `R_AMT` | Requested amount | `"1500.00"` |

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
    
    @action(detail=False, methods=['get'], url_path='fonepay-verify')
    def fonepay_verify(self, request):
        """
        Fonepay redirects here after payment:
        GET /api/orders/fonepay-verify/?PRN=A1B2C3&PS=true&RC=successful
            &UID=123456&DV=...&PID=...&P_AMT=1500.00&R_AMT=1500.00
        """
        # 1. Extract parameters from the callback
        prn = request.query_params.get('PRN')  # Your order code
        pid = request.query_params.get('PID')  # Should match your merchant ID
        ps = request.query_params.get('PS')    # Payment status: "true" or "false"
        rc = request.query_params.get('RC')    # Response code: "successful"
        dv_from_fonepay = request.query_params.get('DV')  # Fonepay's DV hash
        
        # Fonepay Trace ID — this is their unique reference for the transaction
        uid = request.query_params.get('UID')
        p_amt = request.query_params.get('P_AMT')  # Actual amount paid
        r_amt = request.query_params.get('R_AMT')  # Amount you requested

        # 2. Quick check — did the payment succeed on Fonepay's end?
        if ps == 'true' and rc == 'successful':
            
            # 3. SERVER-SIDE VERIFICATION
            # Never trust the callback params alone — verify with Fonepay's server
            conf = settings.FONEPAY_SETTINGS
            
            verify_url = (
                f"{conf['VERIFY_URL']}"
                f"?PID={pid}&PRN={prn}&AMT={r_amt}&BID={uid}&MD=P"
            )
            
            response = requests.get(verify_url)
            
            # 4. If Fonepay's server confirms, update the order
            if "success" in response.text.lower():
                order = get_object_or_404(Order, order_code=prn)
                order.payment_status = "Completed"
                order.transaction_id = uid  # Store Fonepay's Trace ID
                order.save()
                
                return Response({"message": "Fonepay verification successful"})

        # 5. Payment failed or was canceled
        return Response(
            {"error": "Fonepay verification failed or was canceled"},
            status=400
        )
```

### Security considerations for production:

In a production system, you should also:

1. **Verify the DV hash** — Re-calculate the HMAC-SHA512 on the response parameters and compare with `dv_from_fonepay`.
2. **Verify the amount** — Ensure `P_AMT` matches your order's `total_amount`.
3. **Verify the PID** — Ensure it matches your `MERCHANT_ID`.
4. **Add idempotency** — Don't process the same callback twice.

```python
# Enhanced security: verify amount and merchant
order = get_object_or_404(Order, order_code=prn)

# Verify amount matches
expected_amount = "{:.2f}".format(float(order.total_amount))
if r_amt != expected_amount:
    return Response({"error": "Amount mismatch"}, status=400)

# Verify merchant ID
if pid != conf["MERCHANT_ID"]:
    return Response({"error": "Invalid merchant"}, status=400)

# Prevent double-processing
if order.payment_status == "Completed":
    return Response({"message": "Payment already verified"})
```

---

## Step 7 — URL Configuration

Wire up the Fonepay callback:

```python
# orders/urls.py

from django.urls import path, include
from rest_framework.routers import SimpleRouter
from .views import OrderViewSet, PaymentCallbackViewSet

router = SimpleRouter()
router.register(r'', OrderViewSet, basename='order')

urlpatterns = [
    # Fonepay callback — must match your SUCCESS_URL path
    path('fonepay-verify/', PaymentCallbackViewSet.as_view({'get': 'fonepay_verify'}), name='fonepay-verify'),
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

This gives you: `GET /api/orders/fonepay-verify/` — this **must** match the path segment of your `SUCCESS_URL` in settings.

> **Common mistake:** Setting `SUCCESS_URL` to `/api/payments/fonepay-verify/` but registering the URL under `/api/orders/`. These must match exactly.

---

## Step 8 — Frontend Integration

Fonepay is the simplest on the frontend — just redirect to the `full_url`:

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
            payment_method: 'Fonepay'
        })
    });
    
    const data = await response.json();
    
    if (data.payment_method === 'Fonepay' && data.full_url) {
        // Redirect the browser to Fonepay's payment page
        window.location.href = data.full_url;
    }
    
    if (data.error) {
        alert(`Payment failed: ${data.error}`);
    }
};
```

### What the user experiences:

1. User clicks "Pay with Fonepay" → Browser navigates to `https://clientapi.fonepay.com/api/merchantRequest?...`
2. Fonepay shows a bank selection page → User picks their bank (e.g., Global IME)
3. User authorizes payment in their mobile banking app
4. Browser redirects to `https://yoursite.com/api/orders/fonepay-verify/?PRN=...&PS=true&...`
5. Django processes the callback and redirects to a success page

---

## Testing & Sandbox Configuration

### Sandbox vs Production:

| Aspect | Sandbox | Production |
|---|---|---|
| Initiate URL | `https://dev-clientapi.fonepay.com/api/merchantRequest` | `https://clientapi.fonepay.com/api/merchantRequest` |
| Verify URL | `https://dev-clientapi.fonepay.com/api/merchantRequest/verificationMerchant` | `https://clientapi.fonepay.com/api/merchantRequest/verificationMerchant` |
| Credentials | Sandbox PID + Secret Key (from bank) | Production PID + Secret Key (from bank) |

### Getting credentials:

1. Contact your acquiring bank's digital banking department
2. Request Fonepay merchant integration credentials
3. Ask specifically for **sandbox** credentials if you want to test without real money
4. You'll receive: PID (Merchant Code), Secret Key, and environment details

> **Note:** Not all banks provide sandbox access. Some provide production credentials directly with a low-value test mode. Confirm with your bank which environment your credentials belong to.

### Testing the signature independently:

Create a standalone test script to verify your HMAC implementation:

```python
# test_fonepay_signature.py
import hmac
import hashlib
import urllib.parse

PID = "your_pid_here"
SECRET_KEY = "your_secret_here"
PRN = "TEST001"
AMT = "1.00"
DT = "04/16/2026"
R1 = f"Order_{PRN}"
R2 = "N/A"
RU = "https://your-ngrok-url.ngrok-free.dev/api/orders/fonepay-verify/"

# Build signature string
data = f"{PID},P,{PRN},{AMT},NPR,{DT},{R1},{R2},{RU}"
print(f"Signature input: {repr(data)}")

# Hash it
dv = hmac.new(
    SECRET_KEY.encode(),
    data.encode(),
    hashlib.sha512
).hexdigest().upper()

print(f"DV: {dv}")
print(f"DV length: {len(dv)} (expected: 128)")

# Build URL
params = {
    "PID": PID, "MD": "P", "PRN": PRN, "AMT": AMT,
    "CRN": "NPR", "DT": DT, "R1": R1, "R2": R2,
    "RU": RU, "DV": dv
}
url = f"https://clientapi.fonepay.com/api/merchantRequest?{urllib.parse.urlencode(params)}"
print(f"\nFull URL:\n{url}")
```

### Using ngrok for local development:

```bash
# Install ngrok (https://ngrok.com)
ngrok http 8000

# Copy the HTTPS URL (e.g., https://abc123.ngrok-free.dev)
# Update your SUCCESS_URL:
# "SUCCESS_URL": "https://abc123.ngrok-free.dev/api/orders/fonepay-verify/"

# Also add to ALLOWED_HOSTS and CSRF_TRUSTED_ORIGINS:
# ALLOWED_HOSTS = ['abc123.ngrok-free.dev', 'localhost', '127.0.0.1']
# CSRF_TRUSTED_ORIGINS = ['https://abc123.ngrok-free.dev']
```

---

## Production Checklist

Before going live:

- [ ] Use production Fonepay URLs (`clientapi.fonepay.com`, not `dev-clientapi.fonepay.com`)
- [ ] Confirm your PID is activated for the correct environment
- [ ] Update `SUCCESS_URL` to your production HTTPS domain
- [ ] Move `MERCHANT_ID` and `SECRET_KEY` to environment variables
- [ ] Add DV re-verification on the callback response
- [ ] Add amount verification (compare `P_AMT` with your order amount)
- [ ] Add idempotency checks (prevent duplicate payment processing)
- [ ] Ensure your callback URL is registered in `urls.py` and matches `SUCCESS_URL`
- [ ] Add logging for all payment initiation and callback events
- [ ] Handle edge cases: user closes browser during payment, bank timeout, etc.

---

## Common Pitfalls & Debugging

### 1. `{"message":"Error","isSuccess":false}` — HTTP 500 from Fonepay

**Cause:** Your PID is not registered/activated in the Fonepay environment you're hitting.

**Debug:**
- Confirm with your bank whether the PID is for sandbox (`dev-clientapi`) or production (`clientapi`)
- Try switching the `INITIATE_URL` between sandbox and production
- A 500 means Fonepay's **server crashed** looking up your merchant — this is always a credentials/environment issue

### 2. "Invalid Data Validation" — Signature mismatch

**Cause:** The DV hash doesn't match the parameters in the URL.

**Debug:**
```python
# Print the EXACT string being hashed
data = f"{pid},{md},{prn},{amt},NPR,{dt},{r1},{r2},{ru}"
print(repr(data))  # repr() exposes hidden spaces, newlines, encoding issues
```

Common mistakes:
- URL-encoding values before hashing (don't: `04%2F16%2F2026` ✘, use `04/16/2026` ✔)
- Wrong field order (must be `PID,MD,PRN,AMT,CRN,DT,R1,R2,RU`)
- Lowercase hex hash (`bfc4a4...` ✘ → `BFC4A4...` ✔)
- Amount format mismatch (`"1500"` in hash but `"1500.00"` in URL)

### 3. Django `Decimal` formatting issues

**Cause:** Django's `DecimalField` can produce unexpected string representations.

```python
# ❌ Risky — Decimal may format as "1500" or "1.5E+3"
amt = "{:.2f}".format(order.total_amount)

# ✅ Safe — always cast to float first
amt = "{:.2f}".format(float(order.total_amount))
```

### 4. Callback URL mismatch (404 errors)

**Cause:** The `SUCCESS_URL` path doesn't match the URL registered in your `urls.py`.

Common mistake:
```python
# settings.py
"SUCCESS_URL": "https://example.com/api/payments/fonepay-verify/"

# urls.py — registered under /api/orders/, not /api/payments/
path('api/orders/', include('orders.urls'))

# Result: Fonepay redirects to /api/payments/fonepay-verify/ → 404
```

**Fix:** Make sure the path in `SUCCESS_URL` matches where the view is actually mounted.

### 5. Duplicate PRN rejection

**Cause:** Fonepay may reject transactions with a PRN that was used before (even if the previous one failed).

**Fix:** Always use unique order codes. If you need to retry a payment, create a new order with a new code.

### 6. Date format differences

**Cause:** Python's `strftime` on Windows vs Linux can produce different results for some format codes.

```python
# Always use explicit format — safe on all platforms
dt = order.created_at.strftime("%m/%d/%Y")  # "04/16/2026"
```

### 7. R2 field value

**Cause:** Using custom values like `"Zuna_Purchase"` for R2 when Fonepay expects `"N/A"` for unused fields.

**Fix:** Per Fonepay documentation, use `"N/A"` when you don't have meaningful additional information.

---

## Appendix: Complete File Summary

Here's every file you need to create/modify for Fonepay integration:

| File | Purpose |
|---|---|
| `settings.py` | Add `FONEPAY_SETTINGS` configuration block |
| `orders/models.py` | Ensure `Order` model has `transaction_id` and `payment_response` fields |
| `orders/utils.py` | Add `generate_fonepay_signature()` function |
| `orders/payment_strategies.py` | Add `FonepayStrategy` class |
| `orders/views.py` | Add `fonepay_verify` action to `PaymentCallbackViewSet` |
| `orders/urls.py` | Register the `fonepay-verify/` URL path |

---

*Last updated: April 2026. Based on Fonepay Merchant API v2.0.*
