

# nx-router

`nx-router` is a URL routing extension for [Nyxt](https://nyxt.atlas.engineer/). In short, it's an abstraction around Nyxt request resource handlers that introduces the concept of `route` objects.  

Over time, I found the built-in functionality in Nyxt for resource handling becomes difficult to maintain and reason about. I started by using plain handlers to achieve what I needed, but became frustrated with the amount of duplicate logic I found myself writing and the fact I had to come up with everything by myself when I wanted more complex logic.  

`nx-router` is built with the aim of finding a common ground to the most common needs in resource handling. It currently provides five routes: a general-purpose redirector, a general-purpose site blocker, a resource opener, a media toggler, and a `web-route` that ties all of these together for more complex requirements.  

If you're coming from standard browsers, you can think of `nx-router` as similar to existing solutions like [Redirector](https://github.com/einaregilsson/Redirector) and [LeechblockNG](https://github.com/proginosko/LeechBlockNG) put together and on steroids. If you're privacy-minded and already use Nyxt, you might have stumbled upon [nx-freestance-handler](https://github.com/kssytsrk/nx-freestance-handler), an extension akin to [LibRedirect](https://github.com/libredirect/libredirect) which redirects popular sites to their privacy-friendly front-ends. The problem I see with such extensions is they limit the user to  a few predefined sites and create an implicit dependency on their maintainer to update the extension each time one of these goes down or changes its URL structure. See [Examples](#orge0ea95e) for a walk-through on how to set routes up, including how to replicate all of the `nx-freestance-handler` functionality.  


## Installation

To install the extension, you need to download the source and place it in Nyxt's extensions path, given by the value of `nyxt-source-registry` (by default `~/.local/share/nyxt/extensions`).  

    git clone https://git.sr.ht/~conses/nx-router ~/.local/share/nyxt/extensions/nx-router

The extension works with **Nyxt 3 onward** but it's encouraged to use it with the latest version of Nyxt master for the time being.  

If you want to place the extension elsewhere in the system, such as for development purposes, you can configure so via the ASDF source registry mechanism. For this, you'll need to create a file in the source registry directory, `~/.config/common-lisp/source-registry.conf.d/`, and then put the following contents into it, replacing the path with the desired system path.  

    (:tree "/path/to/user/location")

Then, make sure to refresh the ASDF cache via `asdf:clear-source-registry`. ASDF will now be able to find the extension on the custom path. For more information on this utility, please refer to the [ASDF manual](https://asdf.common-lisp.dev/asdf.html).  

By default, Nyxt won't read the custom source registry path we provided, so ensure to include a `reset-asdf-registries` invocation in Nyxt's configuration file too.  

In your Nyxt configuration file, place the following.  

    (define-nyxt-user-system-and-load nyxt-user/router
      :depends-on (nx-router)
      :components ("router.lisp"))

Where `router.lisp` is a custom file that should you should create relative to Nyxt's configuration directory (`*config-file*`'s pathname by default) to provide the extension settings after the `nx-router` system has been successfully loaded. Inside this file, place the following.  

    (define-configuration web-buffer
      ((default-modes `(router:router-mode ,@%slot-value%))))

In addition, you should add the extension options, explained in the following section.  


## Configuration

You must begin by customizing `router-mode`, a mode that takes all of the user routes and applies them to the currently-running browsing session.  

    (define-configuration router:router-mode
      ((router:routes
        (list
         (make-instance 'router:redirector
                        :trigger (match-domain "example.org")
                        :redirect-url "acme.org"
                        :redirect-rule '(("/example" . (not "/" "wiki"))))
         (make-instance 'router:blocker
                        :trigger (match-hostname "www.wonka.inc")
                        :blocklist t)
         (make-instance 'router:media-toggler
                        :trigger (match-url "https://gekko.com/gallery")
                        :instances-builder
                        (make-instance
                         'router:instances-builder
                         :source "https://gekko.com/instances.json"
                         :builder (lambda (instances)
                                    (json:decode-json-from-string
                                     instances))
                        :media-p nil))
         (make-instance 'router:opener
                        :trigger (match-extension "mp3")
                        :resource "mpv --video=no ~a")
         (make-instance 'router:web-route
                        :trigger (match-domain "stark.net")
                        :redirect-rule "https://stark.net/products/(\\w+)/(.*)"
                        :redirect-url (quri:uri "https://\\1.acme.org/\\2")
                        :blocklist '(:path (:ends (not ".html")
                                            :contains "acme"))
                        :resource (lambda (url)
                                    (add-to-csv url))
                        :media-p nil)))))

As you might notice from above, routes can go from really simple ones to ones with really complex logic baked in. As mentioned before, there are currently five predefined routes, with one of them being a combination of the other four.  

All routes derive from a `route` parent class that holds the shared settings. It includes the following slots:  

-   **`trigger`:** the trigger for `route` activation, akin to the predicates used in Nyxt `auto-rules`. One of `match-domain`, `match-host`, `match-regex`, `match-port`, a user-defined function, or a PCRE.
-   **`instances-builder`:** this takes an `instances-builder` object, which in turn takes a source to retrieve instances from and a builder which assembles them into a list.
-   **`toplevel-p` (default: `t`):** whether the `route` is meant to process only top-level requests.

The default settings for the above can be changed in whatever granularity level you desire.  

Due to the nature of WebkitGTK, click events are obscured, which means some requests might not get through, and `iframes` cannot be redirected. To alleviate this, you can control whether only top-level requests should be processed . For example, let's say you decide you want all requests to be processed, including non top-level ones, for all routes by default. You can do so by configuring the `route` user-class in your configuration.  

    (define-configuration router:route
      ((router:toplevel-p nil)))

If you'd like to process non top-level requests only for `redirector` and `opener` routes by default, you can configure them to do so.  

    (define-configuration (router:redirector router:opener)
      ((router:toplevel-p nil)))

Finally, if you'd like to process non top-level requests only for a given instance of a `redirector` class, add the corresponding slot on the `route` instantiation.  

    (make-instance 'router:redirector
                   :trigger (match-domain "example.org")
                   :redirect-url "acme.org"
                   :toplevel-p nil)

`redirector` is a general-purpose redirect `route` that takes the following direct slots:  

-   **`redirect-url`:** a string for a URL host to redirect to or a `quri:uri` object for a complete URL to redirect to. If `redirect-rule` or `trigger` is a PCRE, you can use this as the replacement string and include special sub-strings like those used in `ppcre:regex-replace` (e.g. `\N`, `\'`, `` \` ``, etc.).

-   **`redirect-rule`:** a PCRE to match against the current URL or an association list of redirection rules for paths. If the latter, each entry is a cons of the form `REDIRECT-PATH . TRIGGER-PATHS`, where `TRIGGER-PATHS` is a list of paths from the trigger URL that will be redirected to `REDIRECT-PATH` in the `redirect-url` .  To redirect all paths except `TRIGGER-PATHS` to `REDIRECT-PATH`, prefix this list with `not`.

-   **`original-url`:** takes either a string for the route's original host or a `quri:uri` object for the original complete URL. This is useful for storage purposes (bookmarks, history, etc.) so that the original URL is recorded instead of the redirect's URL.

`blocker` is a general-purpose block `route` that takes the following direct slots:  

-   **`block-banner-p` (default: `t`):** whether to display a block banner upon blocking the `route`.

-   **`blocklist`:** A PCRE to match against the current URL, `t` to block the entire route, or a property list of blocking conditions in the form of `TYPE VALUE`, where `TYPE` is one of `:path` or `:host`.  `VALUE` is another plist of the form `PRED RULES`, where `PRED` is either `:starts`, `:ends`, or `:contains` and `RULES` is a list of strings to draw the comparison against according to the current `TYPE`.  If `RULES` is prefixed with `not`, the entire route will be blocked except for the specified `RULES`.  
    
    You can also pass an integer as `VALUE` to indicate the number of URL *sections* (e.g. `https://example.com/<section1>/<section2>`) to block in case the blocking condition value is not known. Combined `RULES` (specified via `:or`) allow you to specify two or more predicates that you wish to draw the path comparison against, useful if you want to specify a more general block rule first and bypass it for certain scenarios.

`opener` is a general-purpose resource-opener `route` that instructs resources to be opened externally. It takes the following direct slots:  

-   **`resource`:** a resource can be either a function form, in which case it takes a single parameter URL and can invoke arbitrary Lisp forms with it.  If it's a string, it runs the specified command via `uiop:run-program` with the current URL as argument, and can be given in a `format`-like syntax.

`media-toggler` is a `route` that toggles the display status of media elements in the page. It takes the following direct slots:  

-   **`media-p`:** whether to display media in the `route`.

As previously mentioned, `web-route` is a `route` that combines all of the aforementioned routes and takes all their slots. It's helpful when you want to invoke lots of behavior types inside a single `route`.  


## Examples

Set up all Instagram requests to redirect to the hostname `www.picuki.com` and additionally redirect all the paths that don't start with `/`, `/p/`, or `/tv/` to `/profile/` paths, and all paths that do start with `/p/` to `/media/`. This goes in line with the URL structure that Picuki uses. Additionally, block all the paths that don't contain the sub-string `/video`.  

    (make-instance 'router:web-route
                   :trigger (match-regex "https://(www.)?insta.*")
                   :redirect-url "www.picuki.com"
                   :redirect-rule '(("/profile/" . (not "/" "/p/" "/tv/"))
                                    ("/media/" . "/p/"))
                   :blocklist '(:path (:contains (not "/video"))))

Redirect all TikTok requests except the index path, videos, or usernames to `/@placeholder/video/`. This is what [ProxiTok](https://github.com/pablouser1/ProxiTok) uses to proxy TikTok shared links.  

    (make-instance 'router:redirector
                   :trigger (match-domain "tiktok.com")
                   :redirect-url "proxitok.herokuapp.com"
                   :redirect-rule '(("/@placeholder/video/" . (not "/" "/@" "/t"))))

Redirect all Reddit requests to your preferred [Teddit](https://codeberg.org/teddit/teddit) instance and additionally block all of the paths belonging to this route except those that contain the `/comments` sub-string. This would essentially limit you to only being able to access Teddit publications and not being able to access URL sections such as the main feed.  

    (make-instance 'router:web-route
                   :trigger (match-domain "reddit.com")
                   :redirect-url "teddit.namazso.eu"
                   :original-url "www.reddit.com"
                   :blocklist '(:path (:contains (not "/comments"))))

As shown above, you can pass an `:original-url` slot to the `redirector` route so that if you wrap Nyxt internal methods like shown below, history entries will get recorded with the original URL, meaning an inverse redirection will be applied to figure out the original URL structure.  

    (defmethod nyxt:on-signal-load-finished :around ((mode nyxt/history-mode:history-mode) url)
      (call-next-method mode (router:trace-url url)))

Use a `redirector` route with a PCRE trigger to redirect all Fandom routes to your preferred [BreezeWiki](https://breezewiki.com/) instance.  

    (make-instance 'router:redirector
                   :trigger "https://([\\w'-]+)\\.fandom.com/wiki/(.*)"
                   :redirect-url "https://breezewiki.com/\\1/wiki/\\2")

Match YouTube video URLs, videos hosted on its alternative front-ends such as [Invidious](https://invidious.io/), and MP3 files, redirecting all of these to `youtube.com`, and dispatching an `opener` route that invokes an external program with the matched URL, in this case launching an [mpv](https://mpv.io/) player IPC client process through [mpv.el](https://github.com/kljohann/mpv.el) to control the player from Emacs. You can also pass a one-placeholder format string such as `mpv --video=no ~a` to the `:resource` slot if you'd rather not use a Lisp form, where `~a` represents the matched URL. Note how a route's trigger can also consist of a list of predicates for which to match URLs, meaning that on the route below it will match either URLs that contain the video regexp or those that contain the `.mp3` file extension.  

    (make-instance 'router:web-route
                   :trigger '((match-regex ".*/watch\\?v=.*")
                              (match-file-extension "mp3"))
                   :redirect-url "youtube.com"
                   :resource (lambda (url)
                               (eval-in-emacs
                                `(mpv-start ,url))))

The route below provides an `:instances-builder` slot, which takes an `instances-builder` object that can be customized by the user to generate a list of instances. This is only useful if the service provider of `:redirect-url`  hosts an external endpoint where these are stored. Instances will be added to the route's `:trigger` on route instantiation. `:redirect-url` slots can also take an arbitrary function which will compute the redirect hostname to use. See [instances.lisp](instances.lisp) for a few predefined builders for common alternative front-end providers.  

    (defun set-invidious-instance ()
      "Set the primary Invidious instance."
      (let ((instances
              (remove-if-not
               (lambda (instance)
                 (and (string= (alex:assoc-value (second instance)
                                                 :region)
                               "DE")
                      (string= (alex:assoc-value (second instance)
                                                 :type)
                               "https")))
               (json:with-decoder-simple-list-semantics
                 (json:decode-json-from-string
                  (dex:get
                   "https://api.invidious.io/instances.json"))))))
        (first (car instances))))
    
    (make-instance 'router:web-route
                   :trigger (match-domain "youtube.com" "youtu.be")
                   :blocklist '(:path (:starts "/c/"))
                   :redirect-url 'set-invidious-instance
                   :instances-builder
                   ;; or router:invidious-instances-builder
                   (make-instance
                    'instances-builder
                    :source "https://api.invidious.io/instances.json"
                    :builder
                    (lambda (instances)
                      (mapcar
                       'first
                       (json:with-decoder-simple-list-semantics
                         (json:decode-json-from-string
                          instances))))))

In case you'd like to specify a different URL scheme than HTTPS or a different port for `:redirect-url`, you should supply a redirect in the form of a `quri:uri` object. For instance, the following sets up a `redirector` route that redirects Google search results to a locally-running instance of [Whoogle](https://github.com/benbusby/whoogle-search), and where results will appear as if they were searched in Google.  

    (make-instance 'router:redirector
                   :trigger (match-regex "https://whoogle.*" "https://.*google.com/search.*")
                   :redirect-url (quri:uri "http://localhost:5000")
                   :original-url (quri:uri "https://www.google.com"))

If you'd like to randomize `:redirect-url` to a list of the healthiest instances, you can use a service like [Farside](https://sr.ht/~benbusby/farside/). Then, include the following `redirector` route, which redirects all Twitter URLs to the corresponding Farside endpoint.  

    (make-instance 'router:redirector
                   :trigger (match-domain "twitter.com")
                   :redirect-url "farside.link"
                   :redirect-rule '(("/nitter/" . "/")))

The following showcases the use of a `blocker` route that prevents the user from accessing Amazon URLs unless their hostname begins with the `smile` sub-string.  

    (make-instances 'router:blocker
                    :trigger (match-domain "amazon.com")
                    :blocklist '(:host (:starts (not "smile"))))

Use a `blocker` route to block certain paths of the `lemmy.ml` route. Namely, block paths that contain the `post` sub-string or paths that start with the `/u/` sub-string. This blocks all publications and user profiles on the site.  

    (make-instance 'router:blocker
                   :trigger (match-domain "lemmy.ml")
                   :blocklist '(:path (:contains "post" :starts "/u/")))

Use a `blocker` route with a combined `:blocklist` path rule for `github.com` requests. Combined path rules (specified via `:or`) allow you to specify two or more predicates that you wish to draw the path comparison against. In the case below, an integer indicates we want to block paths that consist of a single path sub-section (e.g. `https://github.com/<sub-section>`), *or* block all paths except those that contain the `pulls` or `search` sub-strings. This allows you to specify a more general block rule and bypass it for certain scenarios. In the case below, it blocks all single sub-section paths on `github.com` (such as profiles, the marketplace and so on) but at the same time allows you to use GitHub's search engine and see your listed pull requests.  

    (make-instance 'router:blocker
                   :trigger (match-domain "github.com")
                   :blocklist '(:path (:or 1 (:contains (not "pulls" "search")))))

To block one or more triggers, you can use a `blocker` route with a `:blocklist` set to `t`.  

    (make-instance 'router:blocker
                   :trigger (match-domain "timewastingsite1.com"
                                          "timewastingsite2.com")
                   :blocklist t)


## Contributing

You can send feedback, patches, or bug reports to [public@mianmoreno.com](mailto:public@mianmoreno.com).  
