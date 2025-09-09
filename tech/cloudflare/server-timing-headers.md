# Server-Timing Headers

Cloudflare supports server-timing headers out of the box for their RUM
customers. If you are not a RUM customer, you can easily add these headers
using transform rules.

Default for RUM customers:

- `Server-Timing: cfCacheStatus;desc=<MISS|HIT|EXPIRED>`

- `Server-Timing: cfEdge;dur=<# ms>`

- `Server-Timing: cfOrigin;dur=<# ms>`

Using transform rules:

![transform rules](https://cdn.skiddle.id/images/sharex/2025/09/transform-rules.jpeg)

```
concat('cdn-cache; desc=',http.response.headers['cf-cache-status'][0],', edge; dur=',to_string(cf.timings.edge_msec),', origin; dur=',to_string(cf.timings.origin_ttfb_msec))
```

This transform rule exposes:

- `Server-Timing: cdn-cache; desc=<MISS|HIT>`

- `Server-Timing: edge; dur=<# ms>`

- `Server-Timing: origin; dur=<# ms>`


Workers:
Using Cloudflare Workers, you can add values from existing headers such as CF-Cache-Status into server-timing headers. 

Here is an example returning the cache status (hit, miss, revalidate, etc.):

```javascript
/**

 * @param {Response} response

 * @returns {Response}

*/

function addServerTimingHeaders(response, startTime) {

  const serverTiming = [];

  const cfCache = response.headers.get('cf-cache-status');

  if (cfCache) {

    serverTiming.push(`cf_cache;desc=${cfCache}`);

  }

  serverTiming.push(`worker;dur=${Date.now() - startTime}`);

  response.headers.set('Server-Timing', serverTiming.join(', '));

}
```

Source: [How to use Server Timing to get backend transparency from your CDN](https://www.speedcurve.com/blog/server-timing-time-to-first-byte/)