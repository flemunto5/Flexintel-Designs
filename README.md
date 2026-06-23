# Flexintel-Designs
Flexintel Designs is one stop shop for all digital software development in Customer Relationship Management System (CRM). Content Relationship Management System (CRMS) is a power tool to enhance the smooth administration of website applications in accordance with Enterprise Relationship Planning (ERP)Management practices and culture activities. 
I’ll help you build the full Flexintel Designs checkout system inside Laravel, integrating M‑PESA, PesaPal, and the business logic you provided.
The guide covers every layer: database migrations, Eloquent models, payment service classes, controllers, and callback handling – all production‑ready for a Kenyan payments stack.

---

1. Laravel Setup & M‑PESA SDK Installation

```bash
composer create-project laravel/laravel flexintel-checkout
cd flexintel-checkout

# Official Safaricom M‑PESA PHP SDK
composer require safaricom/mpesa

# (Optional) Guzzle is already included in Laravel
```

---

2. Environment Configuration

Add these to your .env file (get credentials from Safaricom Developer Portal and PesaPal):

```env
# M‑PESA
MPESA_CONSUMER_KEY=your_consumer_key
MPESA_CONSUMER_SECRET=your_consumer_secret
MPESA_SHORTCODE=174379
MPESA_PASSKEY=your_passkey
MPESA_ENVIRONMENT=sandbox
MPESA_CALLBACK_URL=https://your-domain.com/api/mpesa/callback

# PesaPal (if needed)
PESAPAL_CONSUMER_KEY=your_key
PESAPAL_CONSUMER_SECRET=your_secret
PESAPAL_ENVIRONMENT=sandbox
PESAPAL_IPN_URL=https://your-domain.com/api/pesapal/ipn
PESAPAL_CALLBACK_URL=https://your-domain.com/checkout/success

APP_URL=https://your-domain.com
```

---

3. Database Schema (Laravel Migrations)

We replicate the exact Prisma schema using Laravel migrations. Create them:

```bash
php artisan make:migration create_orders_table
php artisan make:migration create_order_items_table
php artisan make:migration create_payment_transactions_table
# Add packages, addons, promo_codes as needed (I'll show the core tables)
```

3.1 Packages & Addons (master data)

```php
// migration for packages
Schema::create('packages', function (Blueprint $table) {
    $table->string('id')->primary();           // using cuid, can use UUID
    $table->string('name');
    $table->string('slug')->unique();
    $table->string('tier');                    // 'STARTER','BUSINESS'...
    $table->float('price');                    // KES
    $table->integer('pages')->nullable();
    $table->integer('support_months')->nullable();
    $table->json('features')->nullable();
    $table->boolean('is_active')->default(true);
    $table->timestamps();
});

// addons table similar
```

3.2 Orders table

```php
Schema::create('orders', function (Blueprint $table) {
    $table->string('id')->primary();
    $table->string('order_number')->unique();
    $table->string('user_id')->nullable();
    $table->string('status')->default('PENDING'); // PENDING, PROCESSING, PAID...
    $table->string('payment_method');              // MPESA, CARD, PESAPAL...
    $table->json('billing_data');                  // name, email, phone...
    $table->float('subtotal');
    $table->float('discount_amount')->default(0);
    $table->float('tax_amount')->default(0);       // 16% VAT
    $table->float('total_amount');
    $table->string('mpesa_request_id')->nullable(); // CheckoutRequestID
    $table->string('pesapal_order_tracking_id')->nullable();
    $table->string('payment_reference')->nullable();
    $table->foreignId('promo_code_id')->nullable()->constrained('promo_codes');
    $table->timestamp('paid_at')->nullable();
    $table->timestamps();
});
```

3.3 Order Items

```php
Schema::create('order_items', function (Blueprint $table) {
    $table->string('id')->primary();
    $table->foreignId('order_id')->constrained()->cascadeOnDelete();
    $table->string('item_type');              // 'package' or 'addon'
    $table->string('item_id');                // references package.id or addon.id
    $table->string('name');
    $table->float('price');                   // locked price at time of order
    $table->integer('quantity')->default(1);
    $table->float('total');
});
```

3.4 Payment Transactions

```php
Schema::create('payment_transactions', function (Blueprint $table) {
    $table->string('id')->primary();
    $table->foreignId('order_id')->constrained()->cascadeOnDelete();
    $table->string('transaction_id');          // gateway transaction ID
    $table->float('amount');
    $table->string('currency')->default('KES');
    $table->string('status');                  // pending, success, failed
    $table->string('gateway');
    $table->json('gateway_response')->nullable();
    $table->timestamps();
});
```

Run php artisan migrate after creating all migrations.

---

4. Eloquent Models

Create models for Order, OrderItem, PaymentTransaction, Package, Addon, PromoCode.
Here’s the Order model with relationships:

```php
// app/Models/Order.php
class Order extends Model
{
    public $incrementing = false;
    protected $keyType = 'string';
    protected $casts = [
        'billing_data' => 'array',
        'paid_at'      => 'datetime',
    ];

    public function items()
    {
        return $this->hasMany(OrderItem::class);
    }

    public function transactions()
    {
        return $this->hasMany(PaymentTransaction::class);
    }

    public function promoCode()
    {
        return $this->belongsTo(PromoCode::class);
    }
}
```

---

5. M‑PESA Service (Laravel Wrapper)

We create a dedicated service class that uses the Safaricom\Mpesa\Mpesa instance.

```php
// app/Services/MpesaService.php
namespace App\Services;

use Safaricom\Mpesa\Mpesa;
use Illuminate\Support\Facades\Log;

class MpesaService
{
    protected $mpesa;

    public function __construct()
    {
        $this->mpesa = new Mpesa(
            config('services.mpesa.consumer_key'),
            config('services.mpesa.consumer_secret'),
            config('services.mpesa.environment')
        );
    }

    public function stkPush(string $phone, float $amount, string $accountReference)
    {
        $shortcode = config('services.mpesa.shortcode');
        $passkey   = config('services.mpesa.passkey');
        $callback  = config('services.mpesa.callback_url');

        $phone = $this->normalizePhone($phone);

        $response = $this->mpesa->STKPushSimulation(
            $shortcode,
            $passkey,
            'CustomerPayBillOnline',
            $amount,
            $phone,
            $shortcode,
            $phone,
            $callback,
            $accountReference,
            'Flexintel Designs - Order ' . $accountReference,
            'Remark'
        );

        return [
            'checkoutRequestId'     => $response['CheckoutRequestID'] ?? null,
            'responseCode'          => $response['ResponseCode'] ?? null,
            'responseDescription'   => $response['ResponseDescription'] ?? null,
        ];
    }

    public function queryStatus(string $checkoutRequestId)
    {
        // Implement STK Push Query similarly if needed
    }

    protected function normalizePhone(string $phone): string
    {
        $cleaned = preg_replace('/\D/', '', $phone);
        if (strpos($cleaned, '0') === 0) {
            $cleaned = '254' . substr($cleaned, 1);
        } elseif (strpos($cleaned, '7') === 0) {
            $cleaned = '254' . $cleaned;
        }
        return $cleaned;
    }
}
```

Bind it in AppServiceProvider or a dedicated service provider:

```php
$this->app->singleton(MpesaService::class, function () {
    return new MpesaService();
});
```

Add a config file config/services.php (or a dedicated config/mpesa.php):

```php
// config/services.php
'mpesa' => [
    'consumer_key'    => env('MPESA_CONSUMER_KEY'),
    'consumer_secret' => env('MPESA_CONSUMER_SECRET'),
    'shortcode'       => env('MPESA_SHORTCODE'),
    'passkey'         => env('MPESA_PASSKEY'),
    'environment'     => env('MPESA_ENVIRONMENT', 'sandbox'),
    'callback_url'    => env('MPESA_CALLBACK_URL'),
],
```

---

6. Checkout Controller

This replicates the exact flow from your API route: price verification, discount, tax, order creation, and payment initiation.

```php
// app/Http/Controllers/Api/CheckoutController.php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\Order;
use App\Models\Package;
use App\Models\Addon;
use App\Models\PromoCode;
use App\Services\MpesaService;
use Illuminate\Http\Request;
use Illuminate\Support\Str;

class CheckoutController extends Controller
{
    public function __construct(
        protected MpesaService $mpesa
    ) {}

    public function checkout(Request $request)
    {
        $validated = $request->validate([
            'cartItems'     => 'required|array',
            'billingData'   => 'required|array',
            'paymentMethod' => 'required|string',
            'promoCode'     => 'nullable|string',
        ]);

        $cartItems     = $validated['cartItems'];
        $billingData   = $validated['billingData'];
        $paymentMethod = $validated['paymentMethod'];
        $promoCode     = $validated['promoCode'] ?? null;

        // 1. Verify prices and build order items (server‑side recalculation)
        $subtotal = 0;
        $orderItems = [];

        foreach ($cartItems as $item) {
            $record = null;
            if ($item['type'] === 'package') {
                $record = Package::where('id', $item['id'])->where('is_active', true)->first();
            } elseif ($item['type'] === 'addon') {
                $record = Addon::where('id', $item['id'])->where('is_active', true)->first();
            }
            if (!$record) {
                return response()->json(['error' => "Item {$item['id']} not found or inactive"], 400);
            }

            $price    = $record->price;
            $quantity = $item['quantity'] ?? 1;
            $subtotal += $price * $quantity;

            $orderItems[] = [
                'item_type' => $item['type'],
                'item_id'   => $item['id'],
                'name'      => $record->name,
                'price'     => $price,
                'quantity'  => $quantity,
                'total'     => $price * $quantity,
            ];
        }

        // 2. Apply promo code (same logic)
        $discount = 0;
        $promo = null;
        if ($promoCode) {
            $promo = PromoCode::where('code', strtoupper($promoCode))
                        ->where('is_active', true)
                        ->first();
            if ($promo) {
                $now = now();
                if ($promo->expiry_date > $now &&
                    (!$promo->usage_limit || $promo->used_count < $promo->usage_limit) &&
                    (!$promo->min_order_amount || $subtotal >= $promo->min_order_amount)) {

                    if ($promo->discount_type === 'percentage') {
                        $discount = $subtotal * ($promo->discount_value / 100);
                        if ($promo->max_discount && $discount > $promo->max_discount) {
                            $discount = $promo->max_discount;
                        }
                    } else {
                        $discount = $promo->discount_value;
                    }
                }
            }
        }

        // 3. Tax (16% VAT)
        $taxable = $subtotal - $discount;
        $tax     = $taxable * 0.16;
        $total   = $taxable + $tax;

        // 4. Create order
        $orderNumber = 'FLEX-' . time() . '-' . Str::random(5);
        $order = Order::create([
            'id'             => (string) Str::ulid(),   // or use Str::uuid()
            'order_number'   => $orderNumber,
            'status'         => 'PENDING',
            'payment_method' => $paymentMethod,
            'billing_data'   => $billingData,
            'subtotal'       => $subtotal,
            'discount_amount'=> $discount,
            'tax_amount'     => $tax,
            'total_amount'   => $total,
            'promo_code_id'  => $promo->id ?? null,
        ]);

        // Attach order items
        foreach ($orderItems as $item) {
            $order->items()->create([
                'id'        => (string) Str::ulid(),
                ...$item,
            ]);
        }

        // Increment promo usage
        if ($promo) {
            $promo->increment('used_count');
        }

        // 5. Initiate payment
        $paymentResponse = [];
        switch ($paymentMethod) {
            case 'MPESA':
                $mpesaResult = $this->mpesa->stkPush(
                    $billingData['phone'],
                    $total,
                    $orderNumber
                );

                if (!$mpesaResult['checkoutRequestId']) {
                    // Handle error gracefully
                    return response()->json(['error' => 'Failed to initiate M-PESA payment'], 500);
                }

                $order->update(['mpesa_request_id' => $mpesaResult['checkoutRequestId']]);

                $order->transactions()->create([
                    'id'             => (string) Str::ulid(),
                    'transaction_id' => $mpesaResult['checkoutRequestId'],
                    'amount'         => $total,
                    'currency'       => 'KES',
                    'status'         => 'pending',
                    'gateway'        => 'mpesa',
                    'gateway_response'=> $mpesaResult,
                ]);

                $paymentResponse = [
                    'status'            => 'pending',
                    'message'           => 'STK Push sent to your phone',
                    'checkoutRequestId' => $mpesaResult['checkoutRequestId'],
                ];
                break;

            // You can add PESAPAL, CARD, BANK_TRANSFER etc. similarly
            default:
                $reference = 'REF-' . time();
                $order->update(['payment_reference' => $reference]);
                $paymentResponse = [
                    'status'    => 'pending',
                    'message'   => "Order created. Please complete payment via {$paymentMethod}",
                    'reference' => $reference,
                ];
        }

        return response()->json([
            'success'       => true,
            'orderId'       => $order->id,
            'orderNumber'   => $orderNumber,
            'totalAmount'   => $total,
            ...$paymentResponse,
        ]);
    }
}
```

---

7. M‑PESA Callback Controller

This processes the STK Push result and updates the order & transaction.

```php
// app/Http/Controllers/Api/MpesaCallbackController.php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\Order;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class MpesaCallbackController extends Controller
{
    public function __invoke(Request $request)
    {
        $body = $request->all();
        Log::info('M‑PESA Callback', $body);

        $callback = $body['Body']['stkCallback'] ?? null;
        if (!$callback) {
            return response()->json(['ResultCode' => 1, 'ResultDesc' => 'Invalid callback']);
        }

        $resultCode       = $callback['ResultCode'];
        $checkoutRequestId = $callback['CheckoutRequestID'];

        $order = Order::where('mpesa_request_id', $checkoutRequestId)->first();
        if (!$order) {
            return response()->json(['ResultCode' => 1, 'ResultDesc' => 'Order not found']);
        }

        if ($resultCode == 0) {
            // Success
            $metadata = $callback['CallbackMetadata']['Item'] ?? [];
            $amount = collect($metadata)->where('Name', 'Amount')->first()['Value'] ?? null;
            $mpesaReceipt = collect($metadata)->where('Name', 'MpesaReceiptNumber')->first()['Value'] ?? null;

            $order->update([
                'status'  => 'PAID',
                'paid_at' => now(),
            ]);

            // Update transaction
            $order->transactions()
                ->where('transaction_id', $checkoutRequestId)
                ->update([
                    'status'         => 'success',
                    'transaction_id' => $mpesaReceipt ?: $checkoutRequestId,
                    'gateway_response'=> $body,
                ]);

            // TODO: Send confirmation email

        } else {
            // Failed
            $order->update(['status' => 'FAILED']);
            $order->transactions()
                ->where('transaction_id', $checkoutRequestId)
                ->update([
                    'status'          => 'failed',
                    'gateway_response' => $body,
                ]);
        }

        return response()->json(['ResultCode' => 0, 'ResultDesc' => 'Accepted']);
    }
}
```

---

8. Routes (API)

Define API routes in routes/api.php:

```php
Route::post('/checkout', [\App\Http\Controllers\Api\CheckoutController::class, 'checkout']);
Route::post('/mpesa/callback', \App\Http\Controllers\Api\MpesaCallbackController::class);
```

The callback URL must be publicly accessible. Use ngrok during development.

---

9. Frontend Integration

You can use the same React/Vue components you provided, just point them to the Laravel API endpoint (/api/checkout).
For Blade‑based frontend, call the API via JavaScript or submit a form. The payment status polling works identically.

---

10. Production Checklist

· Set MPESA_ENVIRONMENT=production and use live credentials.
· Ensure your server has a valid SSL certificate (required by Safaricom).
· Register your production callback URL on the Daraja portal.
· Run php artisan migrate on production.
· Consider queueing the callback processing for heavy loads.
· Enable Laravel’s error logging and monitor M‑PESA callbacks.

---

Summary

You now have a full Laravel implementation of the Flexintel Designs checkout system, with:

· Database schema matching the original Prisma design.
· Server‑side price verification to prevent tampering.
· Promo code and tax calculation exactly as specified.
· M‑PESA STK Push using the official SDK, with proper callback handling.
· Extensible architecture – adding PesaPal or other gateways follows the same pattern.

The system is ready for sandbox testing and can go live with minimal changes. If you need help with the PesaPal or card payment service classes, let me know and I’ll provide those too.
