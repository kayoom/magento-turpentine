# Turpentine Technical Notes

This document is intended to serve as a guide to some of Turpentine's more complicated mechanics and as a small historical guide to Turpentine's development.

## In the beginning...

When Turpentine was started, there were a few pre-existing extensions intended to make Magento work with Varnish, notably Phoenix Media's [Pagecache powered by Varnish](http://www.magentocommerce.com/magento-connect/pagecache-powered-by-varnish.html) and Madalin Oprea's [Magneto-Varnish](https://github.com/madalinoprea/magneto-varnish) but we ([Nexcess.net](http://www.nexcess.net/)) didn't find them suitable for our needs for various reasons, so it was decided that we would make our own.

As context for further discussion, let me explain why an extension is even needed for Magento to work with Varnish. Varnish works as a proxy cache, requests are sent from the client to Varnish, which examines the request to see if Varnish can serve the response from its cache, and if not (or the response is not already in Varnish's cache) the request is forwarded on to the backend (Magento running on your webserver of choice). Magento sends the response back to Varnish, Varnish caches it (or not) and then forwards the response back to the client. The problem with Varnish in conjunction with Magento is that cookies are one of the things that makes Varnish consider a request to be un-cacheable (by default), and Magento sends a session cookie with every dynamic response (when the request goes to a PHP script rather than a static asset like an image). Most of Magento does not function, or does not function correctly unless the session cookie uniquely identifies a visitor, thus additional work is needed to get the two working together.

The most obvious and fastest to implement solution is to just let the first request from a client pass through Varnish to the backend and send the response back unmodified which lets the client get the session cookie, then strip the cookie from further requests from the client and caching those responses. This is how Turpentine originally worked, along with many of the other Varnish extensions available for Magento. However this method quickly runs into several problems:

  * You can't serve cached responses once a client takes an action that modifies their session, such as adding something to the cart. So you set another cookie that says "bypass the Varnish cache" when the session is modified. This means the typical experience for a client is a slow first page load, subsequent pages are much faster (since they are cached), then page loads are slow again once they login or add something to the cart.
  * You have to keep a list of what "actions" cause the client to modify their session so the "bypass the Varnish cache" cookie can be set when needed. There are some tricks you can use to guess when this is, but they are not very reliable.
  * Varnish can end up serving cached content from another session if the cache is cleared and a client that already has a session cookie visits the site.

Given these problems, it's clear that a better solution is needed, which is where ESI (Edge Side Includes) come in.

## ESI or: How I learned to stop worrying and love the Cache

ESI was originally created so that CDN providers (like Akamai) could serve their customer's content from cache but still pull in dynamic content as needed. Fortunately, many proxy cache servers saw the benefit in this and also implemented it which is how it came to be in Varnish. ESI essentially lets part(s) of a page be marked that they should be pulled from a separate URL. What this means for Magento is that pages can be cached as a whole, then the dynamic parts replaced with the content specific for the client's session. This ends up being such an improvement over the previous method that it's like going from a rusty tricycle to a [SR-71 Blackbird](http://en.wikipedia.org/wiki/Lockheed_SR-71_Blackbird).

However, actually implementing ESI in Magento is horrendously complicated as Magento doesn't seem to have been designed with rendering just a single block, rather than an entire page, which is needed to serve the actual ESI requests from Varnish. Luckily, Hugues Alary did most of the hard work and made it available in his [Magento-Varnish](https://github.com/huguesalary/Magento-Varnish) extension. The core of how both Magento-Varnish and Turpentine work is:

  1. The extension has a [layout file](https://github.com/nexcess/magento-turpentine/blob/master/app/design/frontend/base/default/layout/turpentine_esi.xml) that tells the extension which blocks should be included via ESI.
  2. The extension waits for the [`core_block_abstract_to_html_before`](https://github.com/nexcess/magento-turpentine/blob/master/app/code/community/Nexcessnet/Turpentine/etc/config.xml#L183) event to trigger during a request, then [examines the block](https://github.com/nexcess/magento-turpentine/blob/master/app/code/community/Nexcessnet/Turpentine/Model/Observer/Esi.php#L191) the event was fired for and checks whether the block should be included via ESI based on the layout file.
  3. If the block should be ESI included, the block's normal template is replaced by a [special template](https://github.com/nexcess/magento-turpentine/blob/master/app/design/frontend/base/default/template/turpentine/esi.phtml) with the ESI include tag that signals Varnish to instead pull the block's content from a separate request. A flag is also added to the request to signal to Varnish that the request should have ESI processing run on it.
  4. The response finishes and is sent to Varnish, Varnish sees the ESI flag and sends a request to get the ESI content, which is then [rendered](https://github.com/nexcess/magento-turpentine/blob/master/app/code/community/Nexcessnet/Turpentine/controllers/EsiController.php#L58) and sent back to Varnish to be included in the original response.

Turpentine and Magento-Varnish differ most significantly on the final step. To render the ESI included block separately from the original request quite a bit of data is needed from the original request. Things like the block's name, what design/theme was active, and (most importantly) the registry keys that the block needs. Magento-Varnish's approach is that all that data should be saved in the cache, then looked up from the cache when the ESI request came back based on the cache key in the ESI request URL. Unfortunately, this can cause problems if Magento's cache and Varnish's cache are not in sync. For example, if the Magento cache is flushed but the Varnish cache is not, it can lead to Varnish requesting ESI blocks that Magento doesn't have the data to render. With Turpentine, I wanted to avoid this so instead all of the data is [encrypted](https://github.com/nexcess/magento-turpentine/blob/master/app/code/community/Nexcessnet/Turpentine/Model/Observer/Esi.php#L241) then just included in the ESI URL itself, neatly avoiding the cache syncing issue. There was [some concern](https://github.com/nexcess/magento-turpentine/blob/master/app/code/community/Nexcessnet/Turpentine/Model/Observer/Esi.php#L266) that the ESI URLs would be too long and cause problems, as the HTTP standard doesn't define a maximum length. Fortunately it seems most webservers and clients use at least a 2048 char limit and in practice there have been no reports of this being an issue.

## I got 99 problems...

As mentioned, Magento isn't really designed to easily facilitate the rendering of a single block, or really work with the proxy cache/ESI model which means a number of workarounds are required to make it work.

### Generating Session Cookies without PHP

Even with the addition of ESI, a client still needs a way to get their session cookie, or else every client would be sharing the same session (which leads to obvious problems). The first request from a client with no session cookie could be passed back to Magento in order to have Magento generate the session token, but we want the highest cache hit rate and least backend traffic possible. Fortunately, Magento isn't very strict about session tokens. As long as they are unique, Magento doesn't seem to care what they look like. So rather than passing back to Magento to get the session token, why not make Varnish simply generate the session token and add it to the request if the client doesn't send one?

It sounds simple enough, but Varnish doesn't actually have any easy way to generate unique session tokens. At first, this seemed like a wash but then I remembered that you can actually write straight C code in the VCL for more advanced functionality, such as generating the session token! So that's what Turpentine does: a small C function is included in the VCL to [generate uuids](https://github.com/nexcess/magento-turpentine/blob/master/app/code/community/Nexcessnet/Turpentine/misc/uuid.c), then that function is used to [make a session cookie](https://github.com/nexcess/magento-turpentine/blob/master/app/code/community/Nexcessnet/Turpentine/misc/version-3.vcl#L58) if no session cookie was sent in the request, and the cookie is [passed back to the client](https://github.com/nexcess/magento-turpentine/blob/master/app/code/community/Nexcessnet/Turpentine/misc/version-3.vcl#L340) in the response so it is sent with future requests.

### Serializing Registry Data

To include the registry data required for ESI block rendering in the ESI URL, Turpentine simply takes a list of the needed registry keys (provided in the layout file) for the block and serializes it using PHP's native [`serialize`](http://us3.php.net/serialize) function which turns stores the data in a string which we can then include in the URL as a simple GET parameter. However, a registry key can be associated with any data type, including whole objects. While `serialize` will typically work on objects there are some edge cases (such as XML documents) that cannot be serialized. In order to accommodate these documents, Turpentine considers some objects to be "[complex registry data](https://github.com/nexcess/magento-turpentine/blob/master/app/code/community/Nexcessnet/Turpentine/Model/Observer/Esi.php#L441)" and handles those separately from the standard PHP types like string, int, simple classes, etc. Complex registry data is considered to be Magento "models" that correspond to database records and have `getId` methods (typically things like products, categories, etc). Then Turpentine can simply serialize the model's class and ID, then load the model with the ID instead of serializing the entire object. This neatly sidesteps the whole issue of finding a way to serialize objects that are unserializable.

### Runtime Events Registration and Class Rewrites

The flash messages block in Magento is another source of problems. Flash messages are the small blocks near the top of pages that show a little message, generally for only a single page load, typically after an "action" like adding something to the cart triggers the "$product was added to your cart" flash message. The main issue with flash messages is that they don't use a template, so they can be handled like regular blocks (by switching templates) and thus require a block rewrite. However, there are cases where the flash messages are handled separately via another extension which loads them via AJAX or some other mechanism that Turpentine's special handling via ESI/AJAX would interfere with, so Turpentine's handling needs to be toggle-able. To ensure that the messages block's behavior is correct when Turpentine's handling is switched off, Turpentine needs to only add the block rewrite at runtime, after checking whether it should be added or not via the config options. Magento has no support for this though, so a workaround is required to add support for it.

To accomplish this, Turpentine includes an [app "shim" class](https://github.com/nexcess/magento-turpentine/blob/master/app/code/community/Nexcessnet/Turpentine/Model/Shim/Mage/Core/App.php) that provides access to Magento's *protected* members, which works thanks to PHP's object model. In PHP, if class `Child` inherits from class `Parent`, then `Child` objects can access their *protected* methods and members that come from class `Parent`, and more importantly the `Child` object can access the *protected* methods and members that come from class `Parent` on **any** object from `Parent` class or that inherits from `Parent`. This means the Turpentine's app shim can access `Mage_Core_Model_App`'s *protected* members to add block rewrites and register for events without including them in the extension's XML files.

### CSRF Form Key Handling

In recent versions of Magento (CE 1.8+ and EE 1.13+), CSRF protection was added to several additional forms, where previously it had only really been used on login form (in the frontend). This has caused problems for quite a few full page cache implementations in addition to Turpentine because it makes the form submission URLs unique to a session which is then cached and, for users that don't have that session/form key, those forms are then broken (i.e. the "Add to Cart" button clicked with the wrong form key simply redirects the client to the homepage rather than actually adding the product to the cart).

This change effectively broke Turpentine for those Magento versions, and finding a solution was not trivial. The [first attempt](https://github.com/nexcess/magento-turpentine/commit/c1b991722e61d92e62ca48824660dfa519150df9) at a fix was to just keep a list of actions that use the form key, and to replace it with the session's correct form key before the action runs. While it worked, it more or less removed the CSRF protection and felt very "hacky", so after some thought a better solution came to mind. Why not have Varnish use ESI to replace the form key inside the links and forms that use it? This keeps the CSRF protection and would have a negligible performance impact as the form key only needs to be requested once and then can be pulled from cache for future requests from that client and session (since the form key is constant for the life of the session). The actual implementation of this presented two hurdles though:

  1. Because the form key is located inside HTML attributes (`href="http://example.com/example/action/form_key/<esi:include src="http://example.com/getFormKey/"/>/"` for example), by default Varnish doesn't see the ESI include tag which completely breaks the buttons and forms that use the form key.
  2. How does Magento know when to generate the actual form key, and not the ESI tag that is replaced by the form key. For example, the form key ESI is needed when generating the "Add to Cart" button, but the actual form key is needed both for both the ESI request to pull in the form key, and when that button is clicked so that it can be compared to the form key that was submitted.

The first is solved by a Varnish config change. Adding `-p esi_syntax=0x2` to Varnish's startup command tells it to look for any ESI tags, even if they're not in properly structured HTML syntax (like the example). The second was more tricky, at first it seemed a static list of actions that use the form key would be needed (like the original fix) that could be used to tell the mage/core session class when to generate the form key, and when to generate the form key ESI tag. However, the solution was staring me in the face: the request itself tells you when you need the actual form key, as it's only needed when the request includes the form key in the GET params or POST data to do the comparison between the actual form key, and the form key sent with the request. Thus we can just [check](https://github.com/nexcess/magento-turpentine/blob/master/app/code/local/Mage/Core/Model/Session.php#L53) if the form key was sent in the request and either generate the real form key, or the ESI tag.

## Closing Time

As you can see, there are a number of interesting "tricks" required to really integrate Varnish and Magento well, and this only covers the most interesting ones. Things like purging stale ESI blocks or cache warming have been left out, though they are arguably just as important as what was covered for a well functioning Varnish extension. Hopefully in the future Magento will make Varnish integration easier, or even better, add native Varnish support.