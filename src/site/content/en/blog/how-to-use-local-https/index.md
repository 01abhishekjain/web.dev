---
title: How to use HTTPS for local development
subhead: Sometimes, you need to run your local development site with HTTPS. Tools and tips to do this safely and quickly.
authors:
  - maudn
date: 2020-12-07
updated: 2020-12-07
hero: hero.jpg
tags:
  - blog
  - security
---

<hr/>

**Outline:**

- [Running your site locally with HTTPS using mkcert (recommended)](<#running-your-site-locally-with-https-using-mkcert-(recommended)>)
- [Running your site locally with HTTPS: other options](#running-your-site-locally-with-https:-other-options)

<hr/>

_In this article, statements about `localhost` are valid for `127.0.0.1` and `[::1]` as well, since they both describe the local computer address, also called "loopback address". Also, to keep things simple, the port number isn't specified._
_So when you see `http://localhost`, read it as `http://localhost:{port}` or `http://127.0.0.1:{port}`._

If your production website uses HTTPS, you want your local development site to behave **like an
HTTPS site** (if your production website doesn't use HTTPS, [make it a priority to switch to HTTPS](/why-https-matters/)).
Most of the time, you can trust `http://localhost` to behave **like an HTTPS site**. But [in some
cases](/when-to-use-local-https), you need to run your site locally with HTTPS. **Let's take a look
at how to do this.**

**⏩ Are you looking for quick instructions, or have you been here before?
Skip to the [Cheatsheet](/#using-mkcert:-cheatsheet).**

## Running your site locally with HTTPS using mkcert (recommended)

To use HTTPS with your local development site and access `https://localhost` or
`https://mysite.example` (custom hostname), you need a [TLS
certificate](https://en.wikipedia.org/wiki/Public_key_certificate#TLS/SSL_server_certificate). But
browsers won't consider just any certificate valid: your certificate needs to be **signed** by an entity that
is trusted by your browser, called a trusted **[certificate authority (CA)](https://en.wikipedia.org/wiki/Certificate_authority)**.

So, what you can do is create a certificate and sign it with a CA that is **trusted locally** by your
device and browser.

One tool that helps you do this in a few commands is
[mkcert](https://github.com/FiloSottile/mkcert). Here is how it works:

<figure class="w-figure">
  <img src="./mkcert.jpg" alt="How mkcert works: diagram">
  <figcaption class="w-figcaption">If you open your locally running site in your browser using HTTPS, your browser will check the
certificate of your local development server. Upon seeing that the certificate has been signed by
the mkcert-generated certificate authority, the browser checks whether it's registered as a trusted
certificate authority. mkcert is listed as a trusted authority, so your browser trusts the
certificate and creates an HTTPS connection.</figcaption>
</figure>

**Note:** mkcert isn't the only way to create a TLS certificate for local development, but it's a good
one. See [other options](#running-your-site-locally-with-https:-other-options).

{% Aside %}
Why we're recommending mkcert and similar tools:

- mkcert is specialized in creating certificates that are **compliant with what browsers consider
  valid certificates**. It stays updated to match requirements and best practices. This is why you
  won't have to run mkcert commands with complex configurations or arguments to generate the right
  certificates!
- mkcert is a cross-platform tool. Anyone on your team can use it.

Many operating systems may include libraries to produce certificates, such as openssl. Unlike mkcert
and similar tools, such libraries may not consistently produce correct certificates, may require
complex commands to be run, and are not necessarily cross-platform.
{% endAside %}

{% Aside 'gotchas' %}
The mkcert we're interested in [this one](https://github.com/FiloSottile/mkcert). ❌ Not [this one](https://www.npmjs.com/package/mkcert).
{% endAside %}

### Caution

{% Banner 'caution' %}

- Never export or share the file `rootCA-key.pem` mkcert creates automatically when you run `mkcert -install`. **An attacker getting hold of this file can create on-path attacks for any site you may be visiting**. They could intercept secure requests from your machine to any site—your bank, healthcare provider, or social networks. If you need to know where `rootCA-key.pem` is located to make sure it's safe, run `mkcert -CAROOT`.
- Only use mkcert for **development purposes**—and by extension, never ask end-users to run mkcert commands.
- Development teams: all members of your team should install and run mkcert **separately** (not store and share the CA and certificate).

{% endBanner %}

### Setup

1.  **Install mkcert (only once).**

    Install mkcert by following the [instructions](https://github.com/FiloSottile/mkcert#installation).
    For example, on macOS:

    ```bash
    brew install mkcert
    brew install nss # if you use Firefox
    ```

1.  **Add mkcert to your local root CAs.**

    In your terminal, run:

    ```bash
    mkcert -install
    ```

    This generates a local certificate authority (CA).
    Your mkcert-generated local CA is only trusted **locally**, on your device.

1.  **Generate a certificate for your site, signed by mkcert.**

    In your terminal, navigate to your site's root directory or whichever directory you'd like the certificates to be located at.

    Then, run:

    ```bash
    mkcert localhost
    ```

    Or if you're using a custom hostname like `mysite.example`:

    ```bash
    mkcert mysite.example
    ```

    The command above does two things:

    - Generates a certificate for the hostname you've specified
    - Lets mkcert—that you've added as a local CA in Step 2—sign this certificate.

    Now, your certificate is ready and signed by a certificate authority your browser trusts locally. You're almost done, but your server doesn't know about your certificate yet!

1.  **Configure your server.**

    You now need to tell your server to use HTTPS (since development servers tend to use HTTP by default) and to use the TLS certificate you've just created.

    How to do this exactly depends on your server. A few examples:

    **👩🏻‍💻 With node:**

    `server.js` (replace `{path/to/certificate...}` and `{port}`):

    ```javascript
    const https = require('https');
    const fs = require('fs');
    const options = {
      key: fs.readFileSync('{path/to/certificate-key-filename}.pem'),
      cert: fs.readFileSync('{path/to/certificate-filename}.pem'),
    };
    https
      .createServer(options, function (req, res) {
        // server code
      })
      .listen({port});
    ```

    **👩🏻‍💻 With [http-server](https://www.npmjs.com/package/http-server):**

    Start your server as follows (replace `{path/to/certificate...}`):

    ```bash
    http-server -S -C {path/to/certificate-filename}.pem -K {path/to/certificate-key-filename}.pem
    ```

    `-S` runs your server with HTTPS, while `-C` and `-K` set the **c**ertificate and **k**ey.

    **👩🏻‍💻 With a React development server:**

    Edit your `package.json` as follows, and replace `{path/to/certificate...}`:

    ```json
    "scripts": {
    "start": "HTTPS=true SSL_CRT_FILE={path/to/certificate-filename}.pem SSL_KEY_KILE={path/to/certificate-key-filename}.pem react-scripts start"
    ```

    For example, if you've created a certificate for `localhost` that is located in your site's root directory as follows:

    ```text
    |-- my-react-app
        |-- package.json
        |-- localhost.pem
        |-- localhost-key.pem
        |--...
    ```

    Then your `start` script should like as follows:

    ```json
    "scripts": {
        "start": "HTTPS=true SSL_CRT_FILE=localhost.pem SSL_KEY_KILE=localhost-key.pem react-scripts start"
    ```

    **👩🏻‍💻 Other examples:**

    - [Angular development server](https://angular.io/cli/serve)
    - [Python](https://blog.anvileight.com/posts/simple-python-http-server/)

    **Note:** Servers may use a different port for HTTPS.

1.  ✨ You're done!
    Open `https://localhost` or `https://mysite.example` in your browser: you're running your site locally with HTTPS.
    You won't see any browser warnings, because your browser trusts mkcert as a local certificate authority.

### Using mkcert: cheatsheet

{% Details %}
{% DetailsSummary %}
Using mkcert: cheatsheet
{% endDetailsSummary %}

To run your local development site with HTTPS:

1.  Set up mkcert.

    If you haven't yet, install mkcert, for example on macOS:

    ```bash
    mkcert -install

    ```

    Check [install mkcert](https://github.com/FiloSottile/mkcert#installation) for Windows and Linux instructions.

    Then, create a local certificate authority:

    ```bash
    mkcert -install
    ```

2.  Create a trusted certificate.

    ```bash
    mkcert {your hostname e.g. localhost or mysite.example}
    ```

    This create a valid certificate (that will be signed by `mkcert` automatically).

3.  Configure your development server to use HTTPS and the certificate you've created in Step 1.
4.  ✨ You're done! You can now access `https://{your hostname}` in your browser, without warnings

{% Banner 'caution' %}

Do this only for **development purposes** and **never export or share** the file `rootCA-key.pem` (if you need to know where this file is located to make sure it's safe, run `mkcert -CAROOT`).

{% endBanner %}

{% endDetails %}

## Running your site locally with HTTPS: other options

### Self-signed certificate

You may also decide to not use a local certificate authority like mkcert, and instead **sign your certificate yourself**.

Beware there are a few pitfalls with this approach:

- Browsers don't trust you as a certificate authority and they'll show warnings you'll need to bypass manually. In Chrome, you may use the flag `#allow-insecure-localhost` to bypass this warning automatically on `localhost`. It feels a bit hacky, because it is.
- Self-signed certificates won't behave in exactly the same way as trusted certificates.
- This is unsafe if you're working in an insecure network.
- It's not necessarily easier or faster than using a local CA.
- If you're not using this technique in a browser context, you may need to disable certificate verification for your server. Omitting to re-enable it in production would be very problematic.

**Note:** [React's](https://create-react-app.dev/docs/using-https-in-development/) and [Vue's](https://cli.vuejs.org/guide/cli-service.html#vue-cli-service-serve) development server HTTPS options create a self-signed certificate under the hood, if you don't specify any certificate. This is quick, but you'll get browser warnings and encounter other pitfalls listed above. Luckily, you can have it all (quick and without pitfalls): as showed in the [mkcert with React example](/how-to-use-local-https/#setup:~:text=With%20a%20React%20development%20server), you can use frontend frameworks' built-in HTTPS option **and** specify a locally trusted certificate created via mkcert or similar.

{% Details %}
{% DetailsSummary %}
Why browsers, by default, don't trust self-signed certificates.
{% endDetailsSummary %}

<figure class="w-figure">
  <img src="./selfSigned.jpg" alt="Why browsers don't trust self-signed certificates: diagram">
  <figcaption class="w-figcaption">If you open your locally running site in your browser using HTTPS, your browser will check the certificate of your local development server. Upon seeing that the certificate has been signed by yourself, the browser checks whether you're registered as a trusted certificate authority. Because you're not, your browser can't trust the certificate; it displays a warning telling you that your connection is not secure. You may proceed at your own risk — if you do, an HTTPS connection will be created.</figcaption>
</figure>

<figure class="w-figure">
  <img src="./warnings.jpg" alt="Screenshots of warnings showed by browsers when a self-signed certificate is used">
    <figcaption class="w-figcaption">Warnings showed by browsers when a self-signed certificate is used.</figcaption>
</figure>
</figure>

{% endDetails %}

### Certificate signed by a regular certificate authority

You may also find techniques based on having an actual certificate authority—not a local one—sign your certificate.

A few things to keep in mind if you're considering using these techniques:

- You'll have more setup work to do than when using a local CA technique like mkcert;
- You need to use a domain name that you control and that is valid. This means that you **can't** use actual certificate authorities for:
  - `localhost` and other domain names that are [reserved](https://www.iana.org/assignments/special-use-domain-names/special-use-domain-names.xhtml), such as `example` or `test`.
  - Any domain name that you don't control.
  - Top-level domains that are not valid. See the [list of valid top-level domains](https://www.iana.org/domains/root/db).

### Reverse proxy

Another option to access a locally running site with HTTPS is to use a [reverse proxy](https://en.wikipedia.org/wiki/Reverse_proxy) such as [ngrok](https://ngrok.com/).

A few points to consider:

- Anyone can access your local development site once you share with them a URL created with a reverse proxy. This can be very handy when demoing your project to clients. Or this can be a downside, if your project is sensitive.
- You may need to consider pricing.
- New [security measures](/cors-rfc1918-feedback/) in browsers may affect the way these tools work.

### Flag

If you're using a custom hostname like `mysite.example`, you can use a flag in Chrome to forcefully consider `mysite.example` secure. Avoid doing this, because:

- You would need to be 100% sure that `mysite.example` always resolves to a local address, otherwise you could leak production credentials.
- You won't be able to debug across browsers with this trick 🙀.

_With many thanks for contributions and feedback to all reviewers and contributors—especially Ryan Sleevi,
Filippo Valsorda, Milica Mihajlija, Rowan Merewood and Jake Archibald. 🙌_

_Hero image background by [@anandu](https://unsplash.com/@anandu) on [Unsplash](https://unsplash.com/photos/pbxwxwfI0B4), edited._
