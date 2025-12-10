# Pay Once

A small payments API designed to be safe to call repeatedly when networks, clients, or servers retry requests. It is intended to sit in between your checkout API and your Payment Service Provider (PSP).

This project is primarily built to learn, practice and demonstrate Go for production-style backend services and payments correctness patterns.

> **Useful Primers**
> These resources should help describe the problem space, motivations, and inspiration behind this project.
> - Sam Newman's talk on [Timeouts, Retries and Idempotency In Distributed Systems](https://www.infoq.com/presentations/distributed-systems-resiliency/)
> - Stripe Engineering blog - [Designing Robust and Predictable APIs with Idempotency](https://stripe.com/blog/idempotency)
> - Airbnb Tech Blog - [Avoiding Double Payments in a Distributed Payments System](https://medium.com/airbnb-engineering/avoiding-double-payments-in-a-distributed-payments-system-2981f6b070bb)

## Demo
```shell
# Create a payment attempt
$ curl -X POST
```

>
> ## Roadmap
> 
> - [ ] smoke - I can run the service and hit one endpoint.
> - [ ] create attempt - I can create an attempt and get an `attempt_id`.
> - [ ] idempotency (create) - I can repeat create safely (same `attempt_id`).
> - [ ] idempotency (mismatch) - same key, different payload gets rejected.
> - [ ] authorise - I can move an attempt from `CREATED` to `AUTHORIZED`.
> - [ ] idempotency (authorise) - I can repeat authorise safely.
> - [ ] capture - I can move `AUTHORIZED` to `CAPTURED`.
> - [ ] timeline - GET tells a clear "this is what happened" story.
> - [ ] concurrency - parallel calls converge safely.
> - [ ] timeout/retry - retries & timeouts converge on a single outcome.
> - [ ] TBC - decide what's most painful, fragile, annoying and iterate.
>

## API Endpoints
```bash
curl -X POST /attempts -H "Idempotency-Key: k1" -d '{...}'
curl -X POST /attempts/{id}/authorize -H "Idempotency-Key: k2"
curl -X POST /attempts/{id}/capture   -H "Idempotency-Key: k3"
curl -X GET  /attempts/{id}
```

### `POST /attempts`

This is the first call and initialises the payment attempt.
```bash
curl -X POST
```

<details>
<summary><strong>Example Response - Attempt 1</strong></summary>

```json
// HTTP/1.1 201 Created
// Content-Type: application/json
// Location: /attempts/pa_01HZY...
// Idempotency-Key: k1

{
  "attempt_id": "pa_01HZY...",
  "order_id": "ord_123",
  "amount": 2599,
  "currency": "GBP",
  "status": "CREATED",
  "capture_method": "manual",
  "created_at": "2025-11-26T12:34:56Z"
}
```

</details>

<details>
<summary><strong>Example Response - Attempt 2 (same key, same body)</strong></summary>

```json
// HTTP/1.1 200 OK
// Content-Type: application/json
// Idempotency-Replayed: true
// Idempotency-Key: k1

{
  // same as above
}
```

</details>

<details>
<summary><strong>Example Response - Attempt 3 (same key, different body)</strong></summary>

```json
// HTTP/1.1 409 Conflict
// Content-Type: application/json

{
  "error": {
    "code": "IDEMPOTENCY_KEY_REUSE",
    "message": "Idempotency-Key was reused with a different request body."
  }
}
```

</details>

### `POST /attempts/{id}/authorize`

Called by the Checkout API after it has created an attempt and is ready to commit the order. Transitions an existing payment attempt from `CREATED` to `AUTHORIZED` by asking the PSP to reserve funds.

```bash
curl -X POST
```

<details>
<summary><strong>Example Response</strong></summary>

```json
// HTTP/1.1 200 OK
// Content-Type: application/json
// Idempotency-Key: k2

{
  "attempt_id": "pa_01HZY...",
  "order_id": "ord_123",
  "amount": 2599,
  "currency": "GBP",
  "status": "AUTHORIZED",
  "capture_method": "manual",
  "authorization": {
    "authorization_id": "auth_01J...",
    "psp": "stripe",
    "psp_reference": "pi_3N...",
    "authorized_at": "2025-11-26T12:35:10Z"
  }
}
```

</details>

### `POST /attempts/{id}/capture`

Called after authorisation to finalise the payment and transfer funds. Transitions an existing payment attempt from `AUTHORIZED` to `CAPTURED` by instructing the PSP to capture the previously authorised funds.
```bash
curl -X POST
```

<details>
<summary><strong>Example Response</strong></summary>

```json
// HTTP/1.1 200 OK
// Content-Type: application/json
// Idempotency-Key: k3

{
  "attempt_id": "pa_01HZY...",
  "order_id": "ord_123",
  "amount": 2599,
  "currency": "GBP",
  "status": "CAPTURED",
  "capture_method": "manual",
  "authorization": {
    "authorization_id": "auth_01J...",
    "psp": "stripe",
    "psp_reference": "pi_3N...",
    "authorized_at": "2025-11-26T12:35:10Z"
  },
  "capture": {
    "capture_id": "cap_01K...",
    "captured_at": "2025-11-26T12:36:00Z"
  }
}
```

</details>
