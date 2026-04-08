# Webhooks — Complete Reference

## Receiving Webhooks

Webhooks are inbound HTTP POST requests from external services (Stripe, GitHub, etc.) triggered by events on their end. They are **not authenticated with your Bearer token** — they use their own signature verification.

### General Pattern

```php
// Routes — NO api_auth filter on webhook endpoints
$routes->post('api/v1/webhooks/stripe', 'StripeController::webhook');
$routes->post('api/v1/webhooks/github', 'GithubController::webhook');
```

### Stripe Webhook Example

```php
<?php
namespace App\Controllers\Api\V1;

use CodeIgniter\HTTP\ResponseInterface;

class StripeController extends BaseApiController
{
    public function webhook(): ResponseInterface
    {
        $payload   = $this->request->getBody();
        $sigHeader = $this->request->getHeaderLine('Stripe-Signature');
        $secret    = env('STRIPE_WEBHOOK_SECRET');

        // Verify the signature — NEVER skip this
        try {
            $event = \Stripe\Webhook::constructEvent($payload, $sigHeader, $secret);
        } catch (\Stripe\Exception\SignatureVerificationException $e) {
            return $this->error('Invalid signature.', 400);
        } catch (\UnexpectedValueException $e) {
            return $this->error('Invalid payload.', 400);
        }

        // Route by event type
        switch ($event->type) {
            case 'payment_intent.succeeded':
                $this->handlePaymentSuccess($event->data->object);
                break;

            case 'payment_intent.payment_failed':
                $this->handlePaymentFailed($event->data->object);
                break;

            case 'customer.subscription.deleted':
                $this->handleSubscriptionCancelled($event->data->object);
                break;

            case 'customer.subscription.updated':
                $this->handleSubscriptionUpdated($event->data->object);
                break;

            case 'invoice.payment_succeeded':
                $this->handleInvoicePaid($event->data->object);
                break;

            case 'invoice.payment_failed':
                $this->handleInvoiceFailed($event->data->object);
                break;

            default:
                // Log unknown events but always return 200
                log_message('info', 'Unhandled Stripe event: ' . $event->type);
        }

        // Always return 200 — anything else causes Stripe to retry
        return $this->success(null, 'Webhook received.');
    }

    private function handlePaymentSuccess(object $paymentIntent): void
    {
        // Update order status, generate tickets, send confirmation email, etc.
        log_message('info', 'Payment succeeded: ' . $paymentIntent->id);
    }

    private function handlePaymentFailed(object $paymentIntent): void
    {
        // Release reserved stock, notify user, etc.
        log_message('warning', 'Payment failed: ' . $paymentIntent->id);
    }

    private function handleSubscriptionCancelled(object $subscription): void
    {
        // Downgrade user, send cancellation email
        log_message('info', 'Subscription cancelled: ' . $subscription->id);
    }

    private function handleSubscriptionUpdated(object $subscription): void
    {
        // Update plan, adjust billing, etc.
        log_message('info', 'Subscription updated: ' . $subscription->id);
    }

    private function handleInvoicePaid(object $invoice): void
    {
        // Record payment, extend access, etc.
        log_message('info', 'Invoice paid: ' . $invoice->id);
    }

    private function handleInvoiceFailed(object $invoice): void
    {
        // Notify user, retry logic, etc.
        log_message('warning', 'Invoice payment failed: ' . $invoice->id);
    }
}
```

### GitHub Webhook Example

```php
<?php
namespace App\Controllers\Api\V1;

use CodeIgniter\HTTP\ResponseInterface;

class GithubController extends BaseApiController
{
    public function webhook(): ResponseInterface
    {
        $payload   = $this->request->getBody();
        $signature = $this->request->getHeaderLine('X-Hub-Signature-256');
        $secret    = env('GITHUB_WEBHOOK_SECRET');

        // Verify signature
        $expected = 'sha256=' . hash_hmac('sha256', $payload, $secret);
        if (!hash_equals($expected, $signature)) {
            return $this->error('Invalid signature.', 400);
        }

        $event = $this->request->getHeaderLine('X-GitHub-Event');
        $data  = json_decode($payload, true);

        switch ($event) {
            case 'push':
                $this->handlePush($data);
                break;
            case 'pull_request':
                $this->handlePullRequest($data);
                break;
            default:
                log_message('info', "Unhandled GitHub event: {$event}");
        }

        return $this->success(null, 'Webhook received.');
    }

    private function handlePush(array $data): void { /* ... */ }
    private function handlePullRequest(array $data): void { /* ... */ }
}
```

### Webhook Rules

1. **Always return `200`** even for unhandled event types. Non-200 responses cause retries.
2. **Always verify the signature** before processing anything.
3. **Process async when possible** — queue heavy work, return `200` immediately.
4. **Log everything** — webhook failures are hard to debug without a trail.
5. **Idempotency** — webhooks can be delivered more than once. Check event IDs.

### Idempotency Check

```php
$eventId = $event->id;
$existing = $this->processedWebhookModel->find($eventId);
if ($existing) {
    return $this->success(null, 'Already processed.');
}
$this->processedWebhookModel->insert([
    'id'           => $eventId,
    'event_type'   => $event->type,
    'processed_at' => date('Y-m-d H:i:s'),
]);
```

---

## Sending Webhooks (Outbound)

If your app needs to notify third parties of events:

```php
<?php
namespace App\Services;

class WebhookService
{
    /**
     * Dispatch a webhook to a subscriber URL.
     */
    public function dispatch(string $url, string $secret, string $event, array $payload): bool
    {
        $body      = json_encode(['event' => $event, 'data' => $payload, 'timestamp' => time()]);
        $signature = hash_hmac('sha256', $body, $secret);

        $client   = \Config\Services::curlrequest();

        try {
            $response = $client->post($url, [
                'headers' => [
                    'Content-Type' => 'application/json',
                    'X-Signature'  => $signature,
                    'X-Event'      => $event,
                ],
                'body'    => $body,
                'timeout' => 10,
            ]);

            return $response->getStatusCode() === 200;
        } catch (\Exception $e) {
            log_message('error', "Webhook dispatch failed to {$url}: {$e->getMessage()}");
            return false;
        }
    }

    /**
     * Dispatch with retry logic.
     */
    public function dispatchWithRetry(string $url, string $secret, string $event, array $payload, int $maxRetries = 3): bool
    {
        for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
            if ($this->dispatch($url, $secret, $event, $payload)) {
                return true;
            }
            log_message('warning', "Webhook retry {$attempt}/{$maxRetries} for {$url}");
            sleep($attempt * 2);  // exponential backoff
        }
        return false;
    }
}
```

### Webhook Subscriber Model

```php
<?php
namespace App\Models;

use CodeIgniter\Model;

class WebhookSubscriberModel extends Model
{
    protected $table         = 'webhook_subscribers';
    protected $primaryKey    = 'id';
    protected $returnType    = 'object';
    protected $allowedFields = ['url', 'secret', 'events', 'active'];
    protected $useTimestamps = true;
}

// Migration
$this->forge->addField([
    'id'         => ['type' => 'INT', 'unsigned' => true, 'auto_increment' => true],
    'url'        => ['type' => 'VARCHAR', 'constraint' => 500],
    'secret'     => ['type' => 'VARCHAR', 'constraint' => 64],
    'events'     => ['type' => 'JSON'],          // ["order.created", "order.updated"]
    'active'     => ['type' => 'TINYINT', 'default' => 1],
    'created_at' => ['type' => 'DATETIME', 'null' => true],
    'updated_at' => ['type' => 'DATETIME', 'null' => true],
]);
```
