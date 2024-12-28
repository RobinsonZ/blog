+++
date = '2024-12-27'
title = 'Using Github Pages as a URL Shortener'
+++

This blog is, of course, not a 100% complete collection of everything I've done. Even ignoring everything that's behind a corporate firewall, a lot of my work is hosted elsewhere and not transcluded here. Like a [project report I wrote for a class at Penn](https://medium.com/@zkislakrobinson/developing-slate-f751be5fa3db), which is hosted on Medium because that's where we had to put it for that class. But I like to keep my URLs aesthetically coherent, particularly on public-facing documents like my [resume](https://robinsonz.me/robinsonz-resume.pdf) (pdf). For any arbitrary piece of work, I want to be able to point people to `robinsonz.me/something`, like it's my personal `bit.ly`.

The problem: as of right now, all my stuff is hosted on [GitHub Pages](https://pages.github.com/), because it's by far the easiest way to put static files on the Internet for free. I don't need to worry about any of the server administration, potential load issues, etc., but I don't get any dynamic logic. So, making `robinsonz.me/something` just give you a 301 isn't possible. Instead, I use meta tags, a tiny bit of client-side JS, and some Webpack templating to easily add new shortlinks to my website as needed.

<!--more-->

## Client-Side Redirection

The fundamental thing we need to do to implement a redirect from `robinsonz.me/something` to `example.com` is to get the former to return an HTML file that looks something like this:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>Redirecting...</title>
    <meta http-equiv="refresh" content="0;url=https://example.com" />
    <script>
      window.location.href = "https://example.com";
    </script>
  </head>
  <body>
    <p>
      If you are not redirected automatically,
      <a href="https://example.com">click here</a>.
    </p>
  </body>
</html>
```

This file tries to redirect you three different ways.

- Using the [meta refresh directive](https://en.wikipedia.org/wiki/Meta_refresh) with a wait time of zero seconds, to immediately trigger a "auto-refresh" of the page and send us to `https://example.com`.
- As a backup in case the browser blocks the meta refresh directive for some reason (or doesn't support it), I also run one line of Javascript that sets the window location to `https://example.com` as soon as the page loads.
- And just in case, if a browser can't or won't run Javascript, I provide a manual link that the user can click.

Now I've got my HTML. So how do I get it up on the internet?

My personal landing page is a [big chunk of HTML and JS](https://github.com/RobinsonZ/robinsonz.github.io), which gets processed by Webpack to add the necessary Tailwind style classes, bundle up the dependencies, and minify the whole thing for improved performance. The intermediate Webpack means that there need not necessarily be a 1:1 relationship between input HTML files and output HTML files. It also means I can't just toss this in `something/index.html` and call it a day. But, as we'll see in a moment, it enables reusability if you do the right things.

My `webpack.config.js` already has this `HtmlWebpackPlugin` in there for the primary page:

```js
module.exports = {
  // ...
  plugins: [
    new HtmlWebpackPlugin({
      template: path.resolve(__dirname, "src/index.html"),
    }),
    // ...
  ],
};
```

(I've omitted a lot of other configuration for brevity, as there's a bunch of unrelated stuff in my `webpack.config.js`. For a full working example of this and other stuff, see my website's [GitHub repo](<(https://github.com/RobinsonZ/robinsonz.github.io)>).)

To make the original redirect work, I just need to put that above HTML in `src/redirect.html` and add another `HtmlWebpackPlugin`:

```js
module.exports = {
  // ...
  plugins: [
    new HtmlWebpackPlugin({
      template: path.resolve(__dirname, "src/index.html"),
    }),
    new HtmlWebpackPlugin({
      template: path.resolve(__dirname, "src/redirect.html"),
      filename: "something/index.html",
      chunks: [], // don't load JS
    }),
    // ...
  ],
};
```

Like the HTML file, there are three parts to this.

- The `template` parameter obviously sets it up to pull from `src/redirect.html`.
- The `filename` parameter tells Webpack to output the result of the compilation (really just minifying the HTML) to `something/index.html` in the output directory.
- `chunks: []` instructs Webpack _not_ to inject the things it normally injects. If this were another webpage, we'd want to also be downloading the whole Javascript bundle, CSS, etc. but I don't need any of that to fire a redirect.

If I run `npm start` and go to `localhost:8080/something`, I see the "If you are not redirected automatically..." text for half a second before bouncing to `example.com`.

An important caveat, before we go any further: a normal 301 redirect is good for SEO and caching, becasue web crawlers will see it and assign any "link credit" from that link to the page it's actually pointing to. Google and other web crawlers won't bother to process the client-side redirect here at all, which means any links to the redirected page are useless for SEO. (They're also slower.) So if you're creating a hyperlink (or getting other people add links on their sites, or bookmarking it), you should link directly to `example.com`, _not_ link to the redirect. In other words, these shortlinks are good for giving out on a one-time basis for brevity, but shouldn't be used anywhere that you don't expect humans to be reading the URL and typing it in by hand.

## Making it Reusable

I don't want to have to copy that HTML file and change the links in three different places every time I want a new shortened URL, though. So, can we templatize it? Yes. Just use the [`templateParameters` parameter](https://github.com/jantimon/html-webpack-plugin?tab=readme-ov-file#options):

```js
// ...
new HtmlWebpackPlugin({
  template: path.resolve(__dirname, "src/redirect.html"),
  filename: "something/index.html",
  chunks: [], // don't load JS
  templateParameters: {
    redirectUrl: "https://example.com",
  },
}),
// ...
```

And then just replace all the instances in the HTML file with `<%= redirectUrl %>`:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>Redirecting...</title>
    <meta http-equiv="refresh" content="0;url=<%= redirectUrl %>" />
    <script>
      window.location.href = "<%= redirectUrl %>";
    </script>
  </head>
  <body>
    <p>
      If you are not redirected automatically,
      <a href="<%= redirectUrl %>">click here</a>.
    </p>
  </body>
</html>
```

This time, I can run `npm run build` to check the output. We'll look at `dist/something/index.html`, and see that the minified HTML looks exactly like our old file (minus the line breaks).

If I want to add a redirect to `google.com` too, it's just another entry in the `plugins` list:

```js
new HtmlWebpackPlugin({
  template: path.resolve(__dirname, "src/redirect.html"),
  filename: "something/index.html",
  chunks: [], // don't load JS
  templateParameters: {
    redirectUrl: "https://example.com",
  },
}),
new HtmlWebpackPlugin({
  template: path.resolve(__dirname, "src/redirect.html"),
  filename: "google/index.html",
  chunks: [], // don't load JS
  templateParameters: {
    redirectUrl: "https://google.com",
  },
}),
```

And the `webpack.config.js` file is just Javascript, so I can actually package it up into a function:

```js
const clientRedirect = (slug, target) =>
  new HtmlWebpackPlugin({
    template: path.resolve(__dirname, "src/redirect.html"),
    filename: `${slug}/index.html`,
    chunks: [], // don't load JS
    templateParameters: {
      redirectUrl: target,
    },
  });

module.exports = {
  // ...
  plugins: [
    // ...
    clientRedirect("something", "https://example.com"),
    clientRedirect("google", "https://google.com"),
    // ...
  ],
};
```

At this point, I can add static redirects to my heart's content, just by adding another `clientRedirect` entry to the `webpack.config.js`.

## Speeding It Up A Trivial Amount

You might wonder why I chose to write the redirect HTML file to `something/index.html`, not `something.html`. The latter would work on GitHub Pages, because [/foo will serve /foo.html if it exists](https://til.simonwillison.net/github/github-pages#user-content-foo-will-serve-content-from-foohtml-if-it-exists); but in that case, `robinsonz.me/something/` will 404. With the `something/index.html` setup, `robinsonz.me/something` will first 301 redirect you to `/something/` and then serve the page; i.e. both `/something` and `/something/` will work fine. But it's marginally slower.

How can we solve this? Just make `clientRedirect` write two output files. We have to tweak the setup a tiny bit: `clientRedirect` now returns a list, which we then incorporate into the plugins list using [spread syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax). And to be extra cute, we'll use `map` instead of copy-pasting the thing twice.

```js
const clientRedirect = (slug, target) =>
  [`${slug}.html`, `${slug}/index.html`].map(
    (outfile) =>
      new HtmlWebpackPlugin({
        template: path.resolve(__dirname, "src/redirect.html"),
        filename: outfile,
        chunks: [], // don't load JS
        templateParameters: {
          redirectUrl: target,
        },
      })
  );

module.exports = {
  // ...
  plugins: [
    // ...
    ...clientRedirect("something", "https://example.com"),
    ...clientRedirect("google", "https://google.com"),
    // ...
  ],
};
```

Note the preceding `...` on each of the clientRedirect calls.

Now, when we build this site and put it on GitHub Pages, both `/something` and `/something/` immediately serve the (client-side)redirect HTML without any (server-side) redirects. This is a pretty trivial speedup in practice. But it's fun!

## Summary

Using this method as a pseudo-URL shortener is not actually a good idea if you control your own backend. It’s slower than a true server-side 301 redirect, and it’s useless for SEO or link equity. But it works quite well for my purposes: all my links stay consistent, it's easy to add new ones, and I'm not relying on any third-party service besides GitHub Pages.

If you found this article particularly helpful or unhelpful, feel free to drop me a line at `z [at] robinsonz [dot] me`.
