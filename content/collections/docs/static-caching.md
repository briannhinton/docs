---
title: 'Static Caching'
template: page
blueprint: page
intro: |
    Nothing loads faster than static pages. Instead of rendering pages dynamically on demand, Statamic can cache static pages and pass routing to Apache or Nginx with reverse proxying.
id: ffa24da8-3fee-4fc9-a81b-fcae8917bd74
---
## Important Preface

Certain features — such as forms with server-side validation or content randomization — don’t work with static page caching. As long as you understand that, you can leverage static caching for maximum performance.

:::tip
You can **alternatively** use the [static site generator](https://github.com/statamic/ssg) to pre-generate and deploy **fully static HTML sites**.
:::

## Caching Strategies

Each caching strategy can be configured independently. Inside `config/statamic/static_caching.php` you will find two pre-configured strategies - one for each supported driver.

``` php
return [
    'strategy' => 'half',

    'strategies' => [
        'half' => [
            'driver' => 'application',
        ],
        'full' => [
            'driver' => 'file',
        ]
    ]
];
```

Set `strategy` to the name of the strategy you wish to use, or `null` to disable static caching completely.

## Application Driver

The application driver will store your cached page content within Laravel's cache. We refer to this as **half measure**.

This will still run every request through a full instance of Statamic but will serve all request data from a pre-rendered cache, speeding up load times often by half or more. This is an easy, one-and-done setting.

``` php
return [
    'strategy' => 'half',

    'strategies' => [
        'half' => [
            'driver' => 'application',
        ]
    ]
];
```

## File Driver

The file driver will generate completely static `.html` pages ready for your web server to serve directly. This means that the HTML files will be loaded before it even reaches PHP.

We refer to this as <mark>full measure</mark>. This is probably the lightning you seek. ⚡️

``` php
return [
    'strategy' => 'full',

    'strategies' => [
        'full' => [
            'driver' => 'file',
            'path' => public_path('static'),
        ]
    ]
];
```

## Server Rewrite Rules

You will need to configure its rewrite rules when using full measure caching. Here are the rules for each type of server.

### Apache

On Apache servers, you can define rewrite rules inside an `.htaccess` file:

``` htaccess
RewriteCond %{DOCUMENT_ROOT}/static/%{REQUEST_URI}_%{QUERY_STRING}\.html -s
RewriteCond %{REQUEST_METHOD} GET
RewriteRule .* static/%{REQUEST_URI}_%{QUERY_STRING}\.html [L,T=text/html]
```

### Nginx

On Nginx servers, you will need to edit your `.conf` files. They are not located within your project, and may be in a slighly different place depending on your server setup.

Some applications like [Laravel Forge](https://forge.laravel.com) may let you edit your `nginx.conf` from within the UI.

``` nginx
location / {
  try_files /static${uri}_${args}.html $uri /index.php?$args;
}
```

### IIS

On Windows IIS servers, your rewrite rules can be placed in a `web.config` file.

``` xml
<rule name="Static Caching" stopProcessing="true">
  <match url="^(.*)"  />
  <action type="Rewrite" url="/static/{R:1}_{QUERY_STRING}.html"  />
</rule>
```

## Excluding Pages

You may add a list of URLs you wish to exclude from being cached. Pages with forms and listings with `sort="random"` are two common examples of pages that don't work properly when cached statically.

``` php
return [
    'exclude' => [
        '/contact',
        '/blog/*',  // Excludes /blog/post-name, but not /blog
        '/news*',   // Exclude /news, /news/article, and /newspaper
    ]
];
```

Query strings will be omitted from exclusion rules automatically, regardless of whether wildcards are used. For example, choosing to ignore `/blog` will also ignore `/blog?page=2`, etc.

## Invalidation

A statically cached page will be served until it is invalidated. You have a several options for how to invalidate your cache.

### Time Limit

When using the application driver, you may specify the `expiry` time in minutes. After this length of time, the next request will be served fresh. By leaving the expiry setting `null`, it will never expire, except when you manually run `php artisan cache:clear`.

**The expiry option is not available when using the file driver.** The generated HTML files will be served before PHP ever gets hit, and there's just nothing we can do about that.

### When Saving

When saving content, the corresponding item’s URL will be flushed from the static cache automatically.

You may also set specific rules for invalidating other pages when content is saved. For example:

``` php
return [
    'invalidation' => [
        'rules' => [
            'collections' => [
                'blog' => [
                    'urls' => [
                        '/blog',
                        '/blog/category/*',
                        '/',
                    ]
                ],
            ],
            'taxonomies' => [
                'tags' => [
                    'urls' => [
                        '/blog',
                        '/blog/category/*',
                        '/',
                    ]
                ]
            ],
            'globals' => [
                'settings' => [
                    'urls' => [
                        '/*'
                    ]
                ]
            ],
            'navigation' => [
                'links' => [
                    'urls' => [
                        '/*'
                    ]
                ]
            ]
        ]
    ]
];
```

#### Explanation

- “when an entry in the blog collection is saved, we should invalidate the /blog page, any pages beginning with /blog/category/, and the home page.”
- “when a term in the tags taxonomy is saved, we should invalidate those same pages”
- “when the settings global set is saved, we invalidate all urls”
- “when the links navigation is saved, we invalidate all urls”

You may add as many collections and taxonomies as you need.

You may also choose to invalidate the entire static cache by specifying `all`.

``` php
return [
    'invalidation' => [
        'rules' => 'all',
    ]
];
```

### By Force

To clear the static file cache you can run `php please static:clear` (and/or delete the appropriate static file locations).

## File Locations

When using the file driver, the static HTML files are stored in the `static` directory of your webroot, but you can change it.

``` php
return [

    'strategies' => [
        'full' => [
            'driver' => 'file',
            'path' => public_path('static'),
        ]
    ]
];
```

You will need to update your appropriate server rewrite rules.


## Multi-Site

When using [multi-site](/multi-site), the path can accept an array of sites to define separate urls and domains, if needed.

``` php
return [

    'strategies' => [
        'full' => [
            'driver' => 'file',
            'path' => [
               'default'    => public_path('static') . '/domain1.com/',
               'default_fr' => public_path('static') . '/domain1.com/fr/',
               'other_site' => public_path('static') . '/domain2.com/',
            ]
        ]
    ]
];
```
### Rewrite Rules

This multi-site example needs modified rewrite rules. 

#### Apache

``` htaccess
RewriteCond %{DOCUMENT_ROOT}/static/%{HTTP_HOST}/%{REQUEST_URI}_%{QUERY_STRING}\.html -s
RewriteCond %{REQUEST_METHOD} GET
RewriteRule .* static/%{HTTP_HOST}/%{REQUEST_URI}_%{QUERY_STRING}\.html [L,T=text/html]
```

#### Nginx

``` nginx
location / {
  try_files /static/${host}${uri}_${args}.html $uri /index.php?$args;
}
```

#### IIS

``` xml
<rule name="Static Caching" stopProcessing="true">
  <match url="^(.*)"  />
  <action type="Rewrite" url="/static/{SERVER_NAME}/{R:1}_{QUERY_STRING}.html"  />
</rule>
```

:::tip
`{SERVER_NAME}` is used here instead of `{HTTP_HOST}` because `{HTTP_HOST}` may include the port.
:::
