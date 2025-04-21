---
title: Hacking Playwright Network Recordings
date: 2025-04-17 00:00:00 +0000
tags: [playwright, testing]
description: Making Playwright’s HAR recordings more flexible and less brittle for automated tests
---

I've been working on a web app that uses Playwright for end-to-end testing. One of its features is the [HAR recorder](https://playwright.dev/docs/mock#recording-a-har-file), which captures network traffic during a test and replays it in future runs. A HAR file is simply a JSON that describes each HTTP request and response. This means:

- No need to mock every request manually

- Tests don't rely on external services being available at the time of running

- Network calls are deterministic and stable

From the Playwright docs:
    HAR replay matches URL and HTTP method strictly. For POST requests, it also matches POST payloads strictly. If multiple recordings match a request, the one with the most matching headers is picked.

Sample HAR file:

```json
{
  "log": {
    "entries": [
      {
        "startedDateTime": "2025-04-17T12:00:00.000Z",
        "time": 0.9,
        "request": {
          "url": "https://api.example.com/data",
          "method": "GET",
          "headers": [{"name": "Accept", "value": "application/json"}]
        },
        "response": {
          "status": 200,
          "statusText": "OK",
          "content": {"size": 1234, "mimeType": "application/json", "text": "{\"data\":\"data\"}"},
          "bodySize": 1234
        }
      }, 
      {
        "startedDateTime": "2025-04-17T12:02:00.000Z",
        "time": 0.2,
        "request": {
          "url": "https://api.example.com/data",
          "method": "POST",
          "headers": [{"name": "Content-Type", "value": "application/json"}],
          "postData": {"mimeType": "application/json", "text": "{\"key\":\"value\"}"}
        },
        "response": {
          "status": 200,
          "statusText": "OK",
          "content": {"size": 1234, "mimeType": "application/json", "text": "{\"key\":\"value\"}"},
          "bodySize": 1234
        }
      }
    ]
  }
}
```

## Where replayFromHar Falls Short

Playwright tries to match each outgoing request with a HAR entry using *strict* criteria - URL, query params, HTTP method, headers, and body.

There are cases where strict matching is too rigid. You may want some leeway, for example when:

- The request includes timestamps, session tokens, unique IDs
- Payload field order / formatting varies (yes even whitespaces are important)
- Unique values are generated per session / request (e.g. request counter)

There is a [JSON RPC use-case](https://github.com/microsoft/playwright/issues/29190) where a request ID includes a random prefix. This breaks strict matching, even though the response would be valid. Similar discussions also transpired in [this feature request](https://github.com/microsoft/playwright/issues/21405).

Mocking manually would certainly give full control over the requests and responses. HAR recording is convenient for testing since traffic can be quickly captured and rerecording when things change also happens smoothly.

### Modify the Request before HAR Matching

You can intercept and modify requests before they are matched against the HAR, interceptors run sequentially in the order opposite to the registration:

```ts
    await page.routeFromHar('recording.har') // happens second, strict match will now pass assuming recording contains default theme
    await page.route('/api', async route => {
        const postData = { ...route.request().postData(), theme: 'default' }
        await route.fallback({ /* TODO: check this code here */ }) // calling fallback while providing overrides turns the interceptors into a pipe of sorts
    }) // happens first
```

However, sometimes providing a magic value is not the most maintainable as the corresponding recorded value could change.

### Rewrite replayFromHar to Allow Loose Matching

```ts
    async function replayFromHarWithExclude(exclude: { requestPath: string, method: 'POST' | 'PUT' | 'PATCH', fields: string[] }[]) {
        const savedHar = /* parsed HAR file here */

        const harMap = savedHar.log.entries.reduce((acc, cur) => {
            acc[`${cur.url} ${cur.method}`] = cur
            return acc
        }, {})

        await page.route('**/*', route => {
            const matchedEntry = harMap[`${route.request().url()} ${route.request().method()}`]
            
            // continue to the next matching handler if current request is not found in the saved entries
            if (!matchedEntry) return route.fallback()

            // ignore if not POST/PUT/PATCH
            if (route.request().method().match(/POST|PUT|PATCH/)) return route.fallback()

            const shouldExclude = exclude.find(e =>
                e.requestPath === route.request().url() &&
                e.method === route.request().method()
            );

            if (!shouldExclude) return route.fulfil({ response: matchedEntry.response });

            // handle fields to exclude
            const recordedBody = JSON.parse(matchedEntry.request.postData.text)
            const receivedBody = JSON.parse(route.request().postData())

            for (const field of matchedRoute.fields) {
                delete recordedBody[field]
                delete receivedBody[field]
            }

            // check if actual body matches recorded
            if (JSON.stringify(recordedBody) !== JSON.stringify(receivedBody)) {
                return route.abort()
            } else return route.fulfil(route.response())
        })
    }
```
This acts like middleware — giving you a chance to tweak request matching before deciding whether to return the saved response.

This loose-matching technique can be extended to headers or query params too, especially if requests include volatile values like session tokens or cache-busters.

There are full-featured packages that take this further such as [playwright-advanced-har](https://github.com/NoamGaash/playwright-advanced-har), and [playwright-network-cache](https://github.com/vitalets/playwright-network-cache) which reimplements the whole flow without using HAR.

## Is This a Good Idea?

The strength of loosening matching rules include:

- convenient rerecording - no need for handcrafting mocks
- gap between production and local testing behaviours is minimal
- quick iteration

Care should be taken on what exactly can be matched. Timestamps are often safe to ignore as most apps don’t rely on the exact value for behaviour. Other fields (e.g. user IDs, configuration options, tokens) might influence the response or application state, and ignoring them could lead to incorrect assumptions or flaky tests. Additionally, adding custom matching logic introduces cognitive and maintenance overhead. You now have to reason about why certain fields are excluded and ensure your exclusions stay correct as the app evolves.

There are other ways of dealing with variability of this type outside of the test scope, such as upstream at the app level.

Client can be configured to use deterministic tokens. e.g: 

```js
    const timestamp = process.env.APP_ENV === 'test' ? 1710000000000 : Date.now();
```

This will work as long as the dynamic field (like a timestamp or token) isn't being validated at the service.

## TL;DR

- HAR recordings make testing fast and reproducible, but strict matching can break occassionally

- Loosening match rules or filtering noisy fields (like timestamps) improves test resilience

- Be cautious: too much flexibility can lead to false confidence in test coverage

- Prefer upstream fixes (e.g. deterministic values) when possible

