# screenshot

*index.js*

```javascript
const playwright = require("playwright");
const express = require("express");
const morgan = require("morgan");

const main = async function () {
  const browser = await playwright.chromium.launch();

  const app = express();

  // Logging
  app.use(morgan("short"));

  app.use(express.static("static"));

  app.get("/api/screenshot", async function (req, res) {
    const context = await browser.newContext();
    context.setDefaultTimeout(5000);

    try {
      if (!req.query.url.includes("http") || req.query.url.includes("file")) {
        res.status(400).send("Bad Request");
        return;
      }

      const page = await context.newPage();

      const params = new URLSearchParams(req.url.slice(req.url.indexOf("?")));
      await page.goto(params.get("url"));

      const buf = await page.screenshot();

      res.header("Content-Type", "image/png").send(buf);
    } catch (err) {
      console.log("[Error]", req.method, req.url, err);
      res.status(500).send("Internal Error");
    } finally {
      await context.close();
    }
  });

  app.listen(80, () => {
    console.log("Listening on port 80");
  });
};

main();
```

やっていることは単純そう。

urlというクエリパラメータに、`http`, `file`が含まれていたら400 Bad request

urlにアクセスして、スクリーンショット取って返却する感じ。

flagの位置を確認すると、サーバ内`/flag`にある。

*Dockerfile*

```text
COPY ./flag.txt /flag.txt
```

`file:///flag.txt`で読みたくなるが、`file`は封じられている。

case sensitiveかどうかが気になり調べてみると、[stack overflow](https://stackoverflow.com/questions/2148603/is-the-protocol-name-in-urls-case-sensitive)になんとなく書いてあった。

一応許容されそう。

ということで、`FILE:///flag.txt`を行ってみる。

結果は400 Bad Request、、なぜ？と思ったが、`http`は含まれていないといけないっぽい。

`FILE:///flag.txt?http`これで良さそう。 => 良かった。

