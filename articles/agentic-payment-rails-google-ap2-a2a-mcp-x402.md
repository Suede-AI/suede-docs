# We Built Agentic Payment Rails to Sell Work to AI Agents

Why Suede AI separates workflow contracts, Google AP2 authorization, A2A, MCP and x402 settlement, and what that makes possible for builders.

We built Suede AI Agent Studio to turn useful workflows into products that other agents can discover, evaluate and buy.

That requires more than putting a price on an API. An agent needs to understand the inputs, inspect the commercial terms, establish permission to spend, invoke the right version and recover when something fails.

We have spent months building that purchasing path: typed workflow contracts, immutable releases, machine-readable discovery, Google AP2 authorization, x402 settlement and durable execution evidence.

The workflow is the product. The surrounding rails give it a machine-readable contract, a discoverable address, explicit commercial terms, controlled execution and evidence of what happened.

The opportunity is substantial for people who know how to solve a specific business problem. The advantage comes from making useful work dependable enough to purchase repeatedly. Here is the architecture we built to do that.

## Start with a unit of work worth buying

A general-purpose agent can reason about a task. A specialized service can package the implementation, domain rules and output format for one bounded job.

Consider our published **PO Match Gate**. It takes `purchaseOrder` and `invoice` inputs and declares structured output for whether they match, a pass-or-hold result, discrepancies and a note. The current listing is payment-enabled at **0.05 USDC per call**. It gives the architecture a concrete unit of work to examine. [PO Match Gate](https://agents.suedeai.ai/a/po-match-gate-mkgu0)

An accounts-payable assistant could use that comparison as one step in a larger workflow. It would be buying the comparison service, not authorizing payment of the supplier's invoice. A human approver still owns that consequential decision.

That distinction makes the product easier to evaluate. There is a defined input, a bounded action, a declared output and a known price. The buyer can decide whether this capability belongs in its process without rebuilding the entire process around our software.

The same product shape can apply to other focused services. The commercial question is whether the result reliably saves more effort than it costs to obtain and check.

## The protocols have different jobs

Agentic commerce becomes easier to reason about when each protocol has a clear responsibility:

- **A2A, or Agent2Agent**, describes and connects independent agents. Agent Cards advertise capabilities and interfaces. It was originally developed by Google and is now a Linux Foundation project. [A2A specification](https://a2a-protocol.org/latest/specification/)
- **MCP, the Model Context Protocol**, exposes tools and context to compatible assistants through a common interface. A tool schema tells the caller what arguments it accepts. [MCP specification](https://modelcontextprotocol.io/specification/2026-07-28)
- **Google's Agent Payments Protocol, AP2**, provides verifiable purchasing authority through mandates and associated evidence. The current v0.2 model uses Checkout Mandates and Payment Mandates. [AP2 specification](https://ap2-protocol.org/ap2/specification/)
- **x402** expresses payment requirements around a resource request using HTTP 402, then carries the signed payment payload and settlement response. [x402 HTTP flow](https://docs.x402.org/core-concepts/http-402)

Suede implements these as distinct surfaces. Eligible published services can be called through MCP against prepaid workspace credit. The direct public paid-call path uses x402 v2. Our optional AP2 profile adds merchant-side authorization before that settlement path. A2A provides agent discovery and invocation. [Suede integration registry](https://agents.suedeai.ai/api/providers)

This separation matters commercially. A builder should be able to package one useful capability without inventing a new product for each buyer interface.

## Discovery needs an executable contract

An agent buyer needs enough information to decide whether a call is appropriate before it spends anything.

That includes the service identity, input and output schemas, availability, endpoint and commercial terms. It also needs the advertised state to be true. A preview, a payment-enabled release and an unavailable service must remain distinguishable.

We expose discovery through well-known documents, including our [A2A Agent Card](https://agents.suedeai.ai/.well-known/agent-card.json), [x402 manifest](https://agents.suedeai.ai/.well-known/x402) and [AP2 profile](https://agents.suedeai.ai/.well-known/ap2.json). The public catalog carries service contracts and current availability. [Service catalog](https://agents.suedeai.ai/api/catalog)

Schemas do practical work here. They make required inputs inspectable, give clients something to validate and make downstream output handling less dependent on guessing what a paragraph means.

We also publish AgentCash-oriented `x-payment-info` metadata in OpenAPI for concrete payment-enabled routes. That gives compatible discovery clients a concrete route, declared price and payment contract to inspect. [Suede OpenAPI](https://agents.suedeai.ai/openapi.json)

The discovery document gets the service into consideration. The contract gives the buyer a basis for choosing it. The result earns the next purchase.

## Freeze what the buyer is buying

A workflow canvas is a development surface. A buyer needs a release.

Our launch path checkpoints an immutable workflow version and promotes that exact version through Test to Live. Public calls resolve the Live deployment rather than whatever happens to be on the mutable canvas. [Launch documentation](https://agents.suedeai.ai/docs/launching)

This decision connects product development directly to payment integrity.

Suppose a buyer inspects a service, receives a quote and authorizes a request. If the underlying workflow silently changes before execution, the buyer may receive a different operation from the one it approved. Version binding gives the purchase a stable referent.

It also improves debugging. When a result is disputed, the useful question is which release ran with which input under which terms. An editable draft cannot answer that question reliably.

Release isolation also lets a builder iterate without moving the ground underneath existing buyers. Development stays flexible. The purchase stays attached to a specific implementation.

## Put purchasing authority in deterministic code

Models are useful for selecting a capability and producing work. Credential trust and payment permission need a different execution boundary.

Our AP2 v0.2 merchant profile issues short-lived, ES256-signed checkout JWTs. The authorization binds the exact request to the immutable service release and payment terms, including the request digest, amount, asset, network, recipient and expiry. This runs as an optional, experimental AP2-to-x402 merchant profile. [Live AP2 profile](https://agents.suedeai.ai/.well-known/ap2.json)

Verification checks schemas, signatures, configured issuer trust, time, audience, nonce, key binding and the disclosed, supported constraints in deterministic code. A token does not get to choose an arbitrary remote key and declare itself trusted. The trust root must come from configuration.

These checks answer different questions. A valid signature identifies a signer. Checkout binding connects the approval to this purchase. Constraint evaluation determines whether the permitted conditions are met. Replay handling prevents the same authorization from being consumed improperly.

AP2 also distinguishes delegation from action authorization, which is essential when an agent acts under authority granted by someone else. [AP2 authorization framework](https://ap2-protocol.org/ap2/agent_authorization/)

We expose a trusted-surface consent flow for explicit approval of an exact merchant quote. Our merchant issues **Checkout Receipts** for its acceptance decision, while the settlement rail supplies payment evidence. Keeping those roles separate preserves the meaning of each receipt. [Implementation report](https://github.com/google-agentic-commerce/AP2/issues/345)

## Keep permission separate from settlement

On the direct paid HTTP path, x402 v2 handles the payment exchange. Our current public merchant rail uses USDC on Base. Publication and payment enablement are separate states. [Live x402 manifest](https://agents.suedeai.ai/.well-known/x402)

At a high level, a compatible purchase looks like this:

1. Discover the service and inspect its contract.
2. Validate the intended input and request the priced resource.
3. Read the HTTP 402 payment requirements.
4. Where AP2 is negotiated, establish and verify the authorization for that exact checkout.
5. Submit the signed x402 payment payload.
6. Verify and settle through the supported payment path.
7. Execute the bound service and retain payment and delivery evidence separately.

In x402 v2, the protocol uses `PAYMENT-REQUIRED`, `PAYMENT-SIGNATURE` and `PAYMENT-RESPONSE` headers. These make the exchange explicit at the HTTP boundary. [x402 v2 migration guide](https://docs.x402.org/guides/migration-v1-to-v2)

A signature is evidence of authorization for the payment action. Settlement establishes the payment outcome. Neither establishes that the service's answer is correct. That last question belongs to fulfillment and evaluation.

## Design the unhappy path before scaling the happy path

Agent buyers will retry. Connections will drop. Verification providers will become unavailable. A payment may settle before execution fails.

Our payment state machine therefore distinguishes authorization, settlement, execution, completion, failure and pending reconciliation. Durable reservations bind mandate and payment-nonce identities before settlement. Exact retries can recover the previous result; changed-body replays must not become new purchases under old authority.

The reserve-and-commit pattern matters because concurrent requests can otherwise both observe an unused authorization. Persistent state must claim that identity before either request performs the irreversible action, then record what actually happened.

An ambiguous settlement response requires particular care. Once a settlement attempt has been submitted, a timeout is insufficient evidence that no money moved. Retrying through another facilitator can turn an availability problem into a duplicate-payment problem. Current x402 documentation likewise describes pending settlement as a reconciliation state. [x402 facilitator documentation](https://docs.x402.org/core-concepts/facilitator)

We preserve uncertainty instead of replacing it with an optimistic retry. We also preserve settled payment evidence when fulfillment fails.

Idempotency, replay protection and receipt-ledger design are where a service becomes operable. A buyer needs an answer to “What happened to my purchase?” even when the answer is temporarily “Settlement is still being reconciled.”

## Spend controls belong at explicit boundaries

A per-call price limit, an operator's execution ceiling and a buyer's recurring allowance each control a different risk. A buyer might approve one small comparison without granting a service the authority to make repeated purchases. The accounting has to preserve that distinction.

Our AP2 profile supports `payment.amount_range` for an individual checkout. Presented `payment.budget` and `payment.agent_recurrence` constraints return `unresolved_constraint`: a fail-closed execution boundary, because cumulative allowances require atomic accounting across distinct purchases. Mandate and nonce reservations prevent duplicate consumption; they serve a different purpose from that shared spending counter. Verification evaluates the disclosed constraint set, while a complete policy must independently establish which constraints are required. Each control has an explicit responsibility. [Suede's public implementation details](https://github.com/google-agentic-commerce/AP2/issues/345), [AP2 Payment Mandates](https://ap2-protocol.org/ap2/payment_mandate/)

This is how we approach spend limits: define the unit being controlled, put enforcement in deterministic code and maintain the state that particular promise requires. A buyer can build on a clear checkout limit and a clear rejection rule.

## The same product can meet buyers in different places

Direct HTTP is one route to market.

Our MCP surface exposes eligible published local services as tools with real input schemas, metered against prepaid workspace credit. A compatible assistant can use that tool interface without pretending MCP itself is a settlement network. [Suede MCP documentation](https://agents.suedeai.ai/docs/mcp)

Our WebMCP storefront lets a compatible browser agent inspect and purchase through the signed-in workspace session. It reuses the credit rail. WebMCP supplies browser tool exposure; payment stays with the application. The browser integration targets the current Community Group draft and compatible browser environments. [WebMCP draft](https://webmachinelearning.github.io/webmcp/)

A2A, MCP, browser tools and paid HTTP serve different caller environments. Keeping the workflow contract coherent across them is the leverage: improve the underlying service once, then make that release accessible through the appropriate supported surface.

## Built over months, with distinct shipping milestones

The foundation of our workflow-to-payment implementation goes back to June 2026. July work hardened settlement handling and moved the implementation onto x402 v2.

The public shipping record then becomes more specific:

- **August 5:** published-service MCP tools.
- **August 9:** the hardened paid-call launch path.
- **August 13:** native A2A interfaces.
- **August 27:** WebMCP storefront and AgentCash discovery metadata.
- **August 29:** activated AP2 merchant signing keys and replay ledger.

Those dated milestones are recorded in our [integration registry](https://agents.suedeai.ai/integrations).

September and October work continued on consent, receipt boundaries, recovery and verification failures. The public AP2 discussion also records the limits we clarified in October. [AP2 implementation discussion](https://github.com/google-agentic-commerce/AP2/issues/345)

This is months of implementation and hardening: workflow publication, paid invocation, additional buyer interfaces, signed authority and recovery. Each layer solves a concrete problem in turning an automation into a purchasable service.

## Where ACP, UCP, MPP and ERC-8004 fit

Beyond our implemented rails, the wider agent-commerce ecosystem offers additional transaction models. Understanding their roles makes the architecture easier to extend without conflating different jobs.

**OpenAI and Stripe's Agentic Commerce Protocol**, also called ACP, covers programmatic commerce and checkout. **Virtuals' Agent Commerce Protocol** uses the same acronym for a different model: commercial jobs with funding, submission and evaluation. A checkout session and an evaluated escrow job have different lifecycles. [Agentic Commerce Protocol architecture](https://www.agenticcommerce.dev/docs/concepts/architecture), [Virtuals ACP](https://os.virtuals.io/acp/overview)

**Universal Commerce Protocol, UCP**, describes commerce capabilities and includes a negotiated AP2 mandates extension. **Machine Payments Protocol, MPP**, introduced by Stripe and Tempo, is another machine-payment approach. These expand the ecosystem of commerce interfaces and payment designs. [UCP AP2 extension](https://ucp.dev/specification/payment/extensions/ap2-mandates/), [Stripe MPP introduction](https://stripe.com/blog/machine-payments-protocol)

**ERC-8004** concerns agent identity, reputation and validation registries. The current draft explicitly excludes payments. Identity can help a buyer investigate a counterparty; it does not establish that a payment settled or a deliverable is correct. [ERC-8004 draft](https://eips.ethereum.org/EIPS/eip-8004)

This protocol map gives builders a way to choose the right commercial shape: a paid API call, a checkout, an evaluated job or an identity layer. Suede's architecture keeps those responsibilities separate so the useful pieces can be composed deliberately.

## Why this positions us to sell to agents

The commercial opportunity is to make specialized work easier to buy and easier to reuse.

A builder with a strong workflow should not need to rebuild discovery, contracts, payment handling, authorization checks and execution evidence for every service. Shared rails let that builder spend more time on the part a buyer values: the result.

For an agent buyer, a well-packaged service reduces integration work. It can inspect the contract, compare the stated cost with its task, select a supported calling path and consume a structured result. The human business behind that agent can retain clearer approval and accountability boundaries.

That is the position we are building toward: a platform where workflows become independently purchasable capabilities, accessible from the environments buyers already use.

The leverage compounds at the product layer. Every useful workflow can reuse the same commercial foundation. Every supported buyer interface gives that workflow another way to be reached. Every improvement to execution and recovery strengthens the purchasing experience across the platform.

That is why we built these rails: to turn specialized work into software that agents can discover, purchase and put to use.

**Have a workflow worth selling? Build and publish it at [SuedeAI.app](https://suedeai.app). Want us to build and operate it with you? Start at [SuedeAI.agency](https://suedeai.agency).**

## Frequently asked questions

### Does AP2 move the money?

AP2 adds verifiable checkout authorization in our optional experimental merchant profile. x402 v2 handles Suede's direct public settlement path.

### Can an MCP tool charge for useful work?

Yes. Suede's eligible published MCP tools meter execution against prepaid workspace credit. MCP provides the tool interface; the application supplies the commercial and accounting behavior.

### How does a workflow become a paid service?

Publish an eligible workflow as an immutable Live release, configure its commercial terms and enable payments. Its current availability and payment terms are then exposed through the supported discovery surfaces.

### Where can I build a workflow for agent buyers?

Use [SuedeAI.app](https://suedeai.app) to build and publish a workflow. For a managed build and ongoing operation, use [SuedeAI.agency](https://suedeai.agency).
