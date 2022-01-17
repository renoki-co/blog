## Cache Eloquent queries in Laravel 8+

When it comes to speeding up your application, caching can be the best thing to achieve. Laravel comes up with cache drivers pre-installed so you can enjoy the experience out-of-the-box. Redis, Memcached or just using local files, Laravel comes packed with this.

This time, we will talk about caching Eloquent queries directly from the models, thus making database caching a breeze!

<hr>

The package can be found on  [GitHub](https://github.com/renoki-co/laravel-eloquent-query-cache), where the documentation will approach all of the main points of the app. However, during this article, you’ll learn just the basics of caching and clearing cache so you can get started before you will dig deeper.

### Installation

The package can be installed through Composer:

```bash
composer require rennokki/laravel-eloquent-query-cache
```

Your models will need the QueryCacheable trait:

```php
use Rennokki\QueryCache\Traits\QueryCacheable;

class Article extends Model
{
    use QueryCacheable;
}
```

### Enable the caching behavior by default

By default, the package does not enable query caching. To achieve this, add the `$cacheFor` variable in your model:

```php
use Rennokki\QueryCache\Traits\QueryCacheable;

class Article extends Model
{
    use QueryCacheable;

    protected $cacheFor = 180; // 3 minutes
}
```

Whenever a query will be triggered, the cache will intervene and in case the cache is empty for that query, it will store it and next time will retrieve it from the caching database; in case it exists, it will retrieve it and serve it, without hitting the database.

```php
// database hit; the query result was stored in the cache
Article::latest()->get();

// database was not hit; the query result was retrieved from the cache
Article::latest()->get();
```

If you simply want to avoid hitting the cache, you can use the `->dontCache()` before hitting the final method.

```php
Article::latest()->dontCache()->firstOrFail();
```

### Enable the caching behavior query-by-query

The alternative is to enable cache query-by-query if caching by default doesn’t sound like a good option to you.

First of all, remove the `$cacheFor` variable from your model if you have any set previously.

On each query, you can call `->cacheFor(...)` to specify that you want to cache that query.

```php
Article::cacheFor(now()->addHours(24))->paginate(15);
```

### Organize better with tags

Some cache storage, like Redis or Memcached, comes up with the support of tagging your keys. This is useful because we can tag our queries in-cache and invalidate the needed cache whenever we want to.

As a simple example, this can be useful if we want to invalidate the articles list cache when one article gets updated.

```php
$articles = Article::cacheFor(60)->cacheTags(['latest:articles'])->latest()->get();
$article = Article::find($id);

$article->update(['title' => 'My new title']);
Article::flushQueryCache(['latest:articles']);
```

The method `flushQueryCache` invalidates only the caches that were tagged with `latest:articles`. If some other queries would have been tagged with tags other than `latest:articles`, they would be kept in the cache.

### Digging deeper

For more about this package, check out the project’s page on  [GitHub](https://github.com/rennokki/laravel-eloquent-query-cache).

## 💸 Sponsorship

Hi, I'm [Alex](https://github.com/rennokki), the founder of [Renoki Co.](https://github.com/renoki-co). I'm thankful for taking your time to read this article, and I hope that it helped you. Developing and maintaining packages and delivering good articles about Laravel, Kubernetes and AWS take a lot of time, but I believe it's a time well spent.

If you support more helpful articles, or you are using one or more Renoki Co. open-source packages in your production apps, in presentation demos, hobby projects, school projects or so, sponsor our work with [Github Sponsors](https://github.com/sponsors/rennokki). 📦
