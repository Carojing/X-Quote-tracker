# Browser Extraction

Use Chrome with the user's logged-in X state whenever possible. X frequently hides quote lists and view counts from anonymous sessions.

## Original Post

Open the status URL and extract the canonical post time from the main post article:

```js
const original = await tab.playwright.evaluate(() => {
  const article = document.querySelector("article");
  const time = article?.querySelector("time");
  const aria = [...(article?.querySelectorAll("[aria-label]") || [])]
    .map((el) => el.getAttribute("aria-label"))
    .filter(Boolean);
  return {
    title: document.title,
    url: location.href,
    datetime: time?.getAttribute("datetime") || null,
    statLabel: aria.find((x) => /views/i.test(x)) || "",
  };
}, undefined, { timeoutMs: 30000 });
```

## Quote Page Loop

Navigate to `<original-url-without-query>/quotes`, then scroll and de-duplicate by quote status URL.

```js
function parseCountText(value) {
  const match = String(value || "").replace(/,/g, "").match(/([0-9]+(?:\.[0-9]+)?)\s*([KMB])?/i);
  if (!match) return 0;
  const unit = match[2]?.toUpperCase();
  const mult = unit === "K" ? 1e3 : unit === "M" ? 1e6 : unit === "B" ? 1e9 : 1;
  return Math.round(Number(match[1]) * mult);
}

function parseStat(label, key) {
  const re = new RegExp(`([0-9][0-9,]*(?:\\.[0-9]+)?\\s*[KMB]?)\\s+${key}`, "i");
  const match = String(label || "").match(re);
  return match ? parseCountText(match[1]) : 0;
}

async function extractVisibleQuotes(tab, originalHandle) {
  return await tab.playwright.evaluate((originalHandle) => {
    return [...document.querySelectorAll("article")].map((article) => {
      const links = [...article.querySelectorAll("a[href]")].map((link) => link.getAttribute("href"));
      const statusHref = links.find((href) =>
        /^\/[^/]+\/status\/\d+$/.test(href || "") &&
        !String(href).toLowerCase().startsWith(`/${originalHandle.toLowerCase()}/`)
      );
      if (!statusHref) return null;

      const times = [...article.querySelectorAll("time")].map((time) => ({
        text: time.textContent,
        datetime: time.getAttribute("datetime"),
      }));
      const aria = [...article.querySelectorAll("[aria-label]")]
        .map((el) => el.getAttribute("aria-label"))
        .filter(Boolean);
      const lines = (article.innerText || "").split("\n").map((line) => line.trim()).filter(Boolean);
      const statLabel = aria.find((x) => /views/i.test(x) && /likes?/i.test(x)) ||
        aria.find((x) => /views/i.test(x)) ||
        "";

      return {
        name: lines[0] || "",
        handle: `@${statusHref.split("/")[1]}`,
        quote_url: `https://x.com${statusHref}`,
        quote_published_at_utc: times[0]?.datetime || null,
        views: 0,
        likes: 0,
        replies: 0,
        reposts: 0,
        stat_label: statLabel,
        text_preview: lines.slice(4, 8).join(" "),
      };
    }).filter(Boolean);
  }, originalHandle, { timeoutMs: 30000 });
}
```

After each extraction pass, parse `stat_label` in the Node context:

```js
quote.views = parseStat(quote.stat_label, "views?");
quote.likes = parseStat(quote.stat_label, "likes?");
quote.replies = parseStat(quote.stat_label, "repl(?:y|ies)");
quote.reposts = parseStat(quote.stat_label, "reposts?");
```

## Stop Conditions

Stop when one of these is true:

- A full scroll pass adds no new quote status URLs.
- The page visibly stops loading more quote posts.
- The request only needs a live checkpoint and the user accepts a partial window.

Always state whether the 24h or 48h window is final based on the original post timestamp and extraction timestamp.
