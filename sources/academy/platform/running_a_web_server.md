---
title: Running a web server on the Apify platform
description: A web server running in an Actor can act as a communication channel with the outside world. Learn how to set one up with Node.js.
sidebar_position: 11
category: apify platform
slug: /running-a-web-server
---

# Running a web server on the Apify platform

**A web server running in an Actor can act as a communication channel with the outside world. Learn how to set one up with Node.js.**

---

Sometimes, an Actor needs a channel for communication with other systems (or humans). This channel might be used to receive commands, to provide info about progress, or both. To implement this, we will run a HTTP web server inside the Actor that will provide:

- An API to receive commands.
- An HTML page displaying output data.

Running a web server in an Actor is a piece of cake! Each Actor run is available at a unique URL (container URL) which always takes the form `https://CONTAINER-KEY.runs.apify.net`. This URL is available in the [**Actor run** object](/api/v2/actor-run-get) returned by the Apify API, as well as in the Apify console.

If you start a web server on the port defined by the **APIFY_CONTAINER_PORT** environment variable (the default value is **4321**), the container URL becomes available and gets displayed in the **Live View** tab in the Actor run console.

For more details, see [the documentation](/platform/actors/development/programming-interface/container-web-server).

## Building the Actor {#building-the-actor}

Let's try to build the following Actor:

- The Actor will provide an API to receive URLs to be processed.
- For each URL, the Actor will create a screenshot.
- The screenshot will be stored in the key-value store.
- The Actor will provide a web page displaying thumbnails linked to screenshots and a HTML form to submit new URLs.

To achieve this we will use the following technologies:

- [Express.js](https://expressjs.com) framework to create the server
- [Puppeteer](https://pptr.dev) to grab screenshots.
- The [Apify SDK](/sdk/js) to access Apify storages to store the screenshots.

Our server needs two paths:

- `/` - Index path will display a page form to submit a new URL and the thumbnails of processed URLs.
- `/add-url` - Will provide an API to add new URLs using a HTTP POST request.

First, we'll import `express` and create an Express.js app. Then, we'll add some middleware that will allow us to receive form submissions.

```js
import { Actor } from 'apify';
import express from 'express';

await Actor.init();

const app = express();

app.use(express.json());
app.use(express.urlencoded({ extended: true }));
```

Now we need to read the following environment variables:

- **APIFY_CONTAINER_PORT** contains a port number where we must start the server.
- **APIFY_CONTAINER_URL** contains a URL under which we can access the container.
- **APIFY_DEFAULT_KEY_VALUE_STORE_ID** is the ID of the default key-value store of this Actor where we can store screenshots.

```js
const {
    APIFY_CONTAINER_PORT,
    APIFY_CONTAINER_URL,
    APIFY_DEFAULT_KEY_VALUE_STORE_ID,
} = process.env;
```

Next, we'll create an array of the processed URLs where the **n**th URL has its screenshot stored under the key **n**.jpg in the key-value store.

```js
const processedUrls = [];
```

After that, the index route is ready to be defined.

```js
app.get('/', (req, res) => {
    let listItems = '';

    // For each of the processed
    processedUrls.forEach((url, index) => {
        const imageUrl = `https://api.apify.com/v2/key-value-stores/${APIFY_DEFAULT_KEY_VALUE_STORE_ID}/records/${index}.jpg`;

        // Display the screenshots below the form
        listItems += `<li>
    <a href="${imageUrl}" target="_blank">
        <img src="${imageUrl}" width="300px" />
        <br />
        ${url}
    </a>
</li>`;
    });

    const pageHtml = `<html>
    <head><title>Example</title></head>
    <body>
        <form method="POST" action="/add-url">
            URL: <input type="text" name="url" placeholder="http://example.com" />
            <input type="submit" value="Add" />
            <hr />
            <ul>${listItems}</ul>
        </form>
    </body>
</html>`;

    res.send(pageHtml);
});
```

And then a second path that receives the new URL submitted using the HTML form; after the URL is processed, it redirects the user back to the root path.

```js
import { launchPuppeteer } from 'crawlee';

app.post('/add-url', async (req, res) => {
    const { url } = req.body;
    console.log(`Got new URL: ${url}`);

    // Start chrome browser and open new page ...
    const browser = await launchPuppeteer();
    const page = await browser.newPage();

    // ... go to our URL and grab a screenshot ...
    await page.goto(url);
    const screenshot = await page.screenshot({ type: 'jpeg' });

    // ... close browser ...
    await page.close();
    await browser.close();

    // ... save screenshot to key-value store and add URL to processedUrls.
    await Actor.setValue(`${processedUrls.length}.jpg`, screenshot, { contentType: 'image/jpeg' });
    processedUrls.push(url);

    res.redirect('/');
});
```

And finally, we need to start the web server.

```js
// Start the web server!
app.listen(APIFY_CONTAINER_PORT, () => {
    console.log(`Application is listening at URL ${APIFY_CONTAINER_URL}.`);
});
```

### Final code {#final-code}

```js
import { Actor } from 'apify';
import express from 'express';

await Actor.init();

const app = express();

app.use(express.json());
app.use(express.urlencoded({ extended: true }));

const {
    APIFY_CONTAINER_PORT,
    APIFY_CONTAINER_URL,
    APIFY_DEFAULT_KEY_VALUE_STORE_ID,
} = process.env;

const processedUrls = [];

app.get('/', (req, res) => {
    let listItems = '';

    // For each of the processed
    processedUrls.forEach((url, index) => {
        const imageUrl = `https://api.apify.com/v2/key-value-stores/${APIFY_DEFAULT_KEY_VALUE_STORE_ID}/records/${index}.jpg`;

        // Display the screenshots below the form
        listItems += `<li>
    <a href="${imageUrl}" target="_blank">
        <img src="${imageUrl}" width="300px" />
        <br />
        ${url}
    </a>
</li>`;
    });

    const pageHtml = `<html>
    <head><title>Example</title></head>
    <body>
        <form method="POST" action="${APIFY_CONTAINER_URL}/add-url">
            URL: <input type="text" name="url" placeholder="http://example.com" />
            <input type="submit" value="Add" />
            <hr />
            <ul>${listItems}</ul>
        </form>
    </body>
</html>`;

    res.send(pageHtml);
});

app.post('/add-url', async (req, res) => {
    const { url } = req.body;
    console.log(`Got new URL: ${url}`);

    // Start chrome browser and open new page ...
    const browser = await Actor.launchPuppeteer();
    const page = await browser.newPage();

    // ... go to our URL and grab a screenshot ...
    await page.goto(url);
    const screenshot = await page.screenshot({ type: 'jpeg' });

    // ... close browser ...
    await page.close();
    await browser.close();

    // ... save screenshot to key-value store and add URL to processedUrls.
    await Actor.setValue(`${processedUrls.length}.jpg`, screenshot, { contentType: 'image/jpeg' });
    processedUrls.push(url);

    res.redirect('/');
});

app.listen(APIFY_CONTAINER_PORT, () => {
    console.log(`Application is listening at URL ${APIFY_CONTAINER_URL}.`);
});
```

When we deploy and run this Actor on the Apify platform, then we can open the **Live View** tab in the Actor console to submit the URL to your Actor through the form. After the URL is successfully submitted, it appears in the Actor log.

With that, we're done! And our application works like a charm :)

The complete code of this Actor is available on its Store [page](https://apify.com/apify/example-web-server/source-code). You can run it there or copy it to your account.
