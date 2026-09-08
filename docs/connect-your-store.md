---
title: "Connect Your Store"
description: "Let a WhatsApp or voice agent answer from your real catalogue and order data instead of guessing. Connect Shopify or WooCommerce in a click, or point the agent at your own backend with no code."
slug: "connect-your-store"
breadcrumb: "Getting Started"
---

# Connect Your Store

Let a WhatsApp or voice agent answer from your real catalogue and order data instead of guessing. Connect Shopify or WooCommerce in a click, or point the agent at your own backend with no code.

:::cards
/docs/agent-tools-mcp | Account MCP Server | wrench | Give an external AI agent tools that act on your account
/docs/whatsapp-setup | WhatsApp Setup | message-circle | Connect a number and go live
:::

## Overview

An agent with no access to your store can only talk in generalities. It will invent a price, promise stock it cannot see, and guess at where an order is. Connecting your store fixes that: the agent looks the answer up at the moment it replies.

There are three routes, and they all end in the same place. Pick the one that matches your site:

| Your site | Route | What it takes |
| --- | --- | --- |
| Shopify | Built-in connector | Sign in, grant access |
| WooCommerce / WordPress | Built-in connector | Paste a read-only REST key |
| Anything else | Custom tools | Expose a few read-only endpoints, then point the agent at them |

All three work for **WhatsApp and voice agents alike**, with one exception noted under [sending product photos](#send-product-photos).

## Shopify

1. Open **Integrations** on your agent.
2. Choose **Shopify**, then **Connect**. You are taken to Shopify to sign in and grant access, then brought back.
3. The Shopify tools are switched on for that agent automatically.

The agent can then look up an order's payment and fulfilment status with tracking, search products for price and stock, find a customer, and start a draft order.

## WooCommerce

WooCommerce uses a read-only API key you create yourself, so there is no app to install on your site.

1. In WordPress, go to **WooCommerce → Settings → Advanced → REST API** and **Add key**. Set **Permissions** to **Read**. Copy the consumer key and consumer secret. WooCommerce shows the secret once.
2. In CallMissed, open **Integrations** on your agent, choose **WooCommerce**, and paste three values: your store URL, the consumer key, and the consumer secret.
3. Press **Connect**. We make one authenticated read to confirm the key works before saving it, and the WooCommerce tools switch on for that agent.

<Callout type="warn">
  Your store URL must be **https**. WooCommerce only accepts key-and-secret authentication over TLS; over plain `http` the credentials would be readable in transit, so we refuse the connection rather than send them in the clear.
</Callout>

The key is stored encrypted and used only on your account's conversations. Reconnecting replaces it.

## Any other site

If you run your own backend (custom, headless, Magento, Django, Rails, a spreadsheet behind an API), expose a few read-only endpoints and register them as tools. Nothing runs on your side but the endpoints themselves: the rest is configuration, in the console or through the [Agent Tools API](/docs/agent-tools).

### Design the endpoints

The agent is a language model, not a browser, so shape these for a model rather than for a web page.

- **Keep responses small.** A tool response is truncated past **32 KB**. Return flat objects with the handful of fields a customer actually asks about, and cap list results at about 5 to 10. Never return a full catalogue document.
- **Return the price the customer will actually pay.** If your storefront displays a tax-inclusive price, return the tax-inclusive number. An agent quoting your pre-tax figure will under-quote every customer.
- **Return absolute URLs.** Both the product page link and the image URL must be complete `https://` addresses, because the agent sends them to the customer as-is.
- **Only return what is published.** Filter out drafts, hidden and inactive items. A public catalogue endpoint often does not, and an agent will happily recommend an unreleased product.
- **Say "not found" in words.** Return something like `{"found": false, "message": "No orders for that number."}` rather than an empty list or a 404. The model relays the message.
- **Authenticate with a header.** A single static key in a header such as `X-API-Key`, compared in constant time, is enough. Restrict the key to letters and digits.

A product search that returns this is doing its job:

```json
{
  "count": 1,
  "products": [
    {
      "id": "kit-104",
      "title": "Line Follower Robot Kit",
      "price": 1180,
      "currency": "INR",
      "inStock": true,
      "url": "https://yourstore.com/product/line-follower",
      "image": "https://yourstore.com/media/line-follower.jpg"
    }
  ]
}
```

### Register the tool

1. Open your agent, go to **Tools**, and choose **New tool**.
2. Give it a **name** and a **description**. The description is the only thing telling the model when to reach for it, so write it as an instruction: "Search the catalogue by keyword. Returns price, stock, the product link and an image URL."
3. Set the **method** and **URL**, for example `GET https://api.yourstore.com/agent-tools/products/search`.
4. Add the parameters:
   - Your API key: a **Header** parameter named `X-API-Key`, source **Secret**. It is encrypted and never shown again.
   - The search term: a **Query** parameter named `q`, source **Agent decides**, described as "what the customer is looking for".
5. **Test** it, then **Save**. Repeat for each endpoint: product search, product detail, order status.

<Callout type="info">
  Your endpoint must be reachable on the public internet over HTTPS. Requests to private, loopback or cloud-metadata addresses are refused, and every redirect is re-checked, so an internal URL cannot be reached through a redirect either.
</Callout>

## Identify the customer safely

Order history is the case where a small mistake matters. If a tool takes the phone number as a value the **agent** supplies, then whatever the customer types becomes the lookup: someone can ask for orders belonging to a number that is not theirs, and the agent will fetch them.

Bind the identity to the conversation instead:

1. On the order-lookup tool, add the parameter that carries the customer's number, for example a **Query** parameter named `phone`.
2. Set its source to **From the conversation**, and pick **Their phone number**.

That value is now filled in by the server from the person actually in the chat. The model never sees the parameter and cannot set it, so a customer asking about someone else's number changes nothing. The same control can bind their name, their email, or the conversation ID.

<Callout type="warn">
  A voice call or a web chat may have no known contact. An unbound value is simply left out, so decide what your endpoint should do: either mark the parameter **required** so the tool fails cleanly rather than running an unscoped query, or have the endpoint return "not found" when the identity is missing.
</Callout>

<Callout type="info">
  **Test** runs outside any conversation, so there is no real customer to bind to. It sends obvious placeholders instead (`+10000000000`, `test@example.com`), which is enough to prove your endpoint is reachable and your key works. Expect it to answer "not found". Check the real behaviour from a live chat.
</Callout>

If you would rather let a customer ask about an order they have the number for, take the order ID **and** a detail only the real customer knows, such as the last four digits of the phone on the order. Return the same "not found" response for a wrong ID and a failed check, so the endpoint cannot be used to discover which order IDs exist.

## Send product photos

A customer asking to see something wants a picture, not a URL.

Switch on the **`send_image`** tool on your agent. When the model has an image URL from your catalogue tool, it sends the photo with a caption. Put the product name, the price and the link in the caption so one message carries everything.

Two requirements: the image URL must be a public `https://` link that needs no login, and it must be a **JPEG or PNG**. WhatsApp fetches the image from your server directly, so a URL behind an auth wall silently fails to send.

<Callout type="info">
  `send_image` is WhatsApp-only. On a voice call the agent will explain the product and can text the link instead.
</Callout>

## Tell the agent how to use it

Connecting tools is half the job. The agent's instructions decide whether it uses them well. Add something like this to your agent's prompt:

```text
You answer for {store name}. Never invent a price, stock level or delivery date:
look it up with your tools, and if a tool returns nothing, say so plainly.

When a customer asks about a product, search the catalogue, then send the photo
with send_image and put the name, price and link in the caption.

When a customer asks about an order, look it up. If you cannot identify them,
ask for their order number, then for the last four digits of the phone on the
order before sharing any detail.
```

## Check it works

1. Open the tool and press **Test** to confirm the endpoint answers and your key is accepted.
2. Message your agent on WhatsApp with a real product name and confirm the price it quotes matches your website exactly.
3. Ask it to show you the product and confirm the photo arrives.
4. Ask about an order from a number that has one, then from a number that does not, and confirm the second says it found nothing.
5. Ask it for orders belonging to a number that is not yours. It should not return them.
