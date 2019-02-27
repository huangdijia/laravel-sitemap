# **Laravel Sitemap package**

Sitemap generator for Laravel

## Installation

*Run the following command:*

```bash
composer require huangdijia/laravel-sitemap
```

*Publish needed assets (styles, views, config files) :*

```bash
php artisan vendor:publish --provider="Huangdijia\Sitemap\SitemapServiceProvider"
```

## Examples

### How to generate dynamic sitemap (with optional caching)

Example: How to create dynamic sitemap

```php
Route::get('sitemap', function() {

    // create new sitemap object
    $sitemap = App::make('sitemap');

    // set cache key (string), duration in minutes (Carbon|Datetime|int), turn on/off (boolean)
    // by default cache is disabled
    $sitemap->setCache('laravel.sitemap', 60);

    // check if there is cached sitemap and build new only if is not
    if (!$sitemap->isCached()) {
        // add item to the sitemap (url, date, priority, freq)
        $sitemap->add(URL::to('/'), '2012-08-25T20:10:00+02:00', '1.0', 'daily');
        $sitemap->add(URL::to('page'), '2012-08-26T12:30:00+02:00', '0.9', 'monthly');

        // add item with translations (url, date, priority, freq, images, title, translations)
        $translations = [
            ['language' => 'fr', 'url' => URL::to('pageFr')],
            ['language' => 'de', 'url' => URL::to('pageDe')],
            ['language' => 'bg', 'url' => URL::to('pageBg')],
        ];
        $sitemap->add(URL::to('pageEn'), '2015-06-24T14:30:00+02:00', '0.9', 'monthly', [], null, $translations);

        // add item with images
        $images = [
            ['url' => URL::to('images/pic1.jpg'), 'title' => 'Image title', 'caption' => 'Image caption', 'geo_location' => 'Plovdiv, Bulgaria'],
            ['url' => URL::to('images/pic2.jpg'), 'title' => 'Image title2', 'caption' => 'Image caption2'],
            ['url' => URL::to('images/pic3.jpg'), 'title' => 'Image title3'],
        ];
        $sitemap->add(URL::to('post-with-images'), '2015-06-24T14:30:00+02:00', '0.9', 'monthly', $images);

        // get all posts from db
        $posts = DB::table('posts')->orderBy('created_at', 'desc')->get();

        // add every post to the sitemap
        foreach ($posts as $post) {
            $sitemap->add($post->slug, $post->modified, $post->priority, $post->freq);
        }
    }

    // show your sitemap (options: 'xml' (default), 'html', 'txt', 'ror-rss', 'ror-rdf')
    return $sitemap->render('xml');
});
```

Example: How to create dynamic sitemap with image tags
In this example the posts model has a belongsToMany relationship to images.

```php
Route::get('sitemap', function() {

    // create new sitemap object
    $sitemap = App::make('sitemap');

    // set cache key (string), duration in minutes (Carbon|Datetime|int), turn on/off (boolean)
    // by default cache is disabled
    $sitemap->setCache('laravel.sitemap', 60);

    // check if there is cached sitemap and build new only if is not
    if (!$sitemap->isCached()) {
        // add item to the sitemap (url, date, priority, freq)
        $sitemap->add(URL::to('/'), '2012-08-25T20:10:00+02:00', '1.0', 'daily');
        $sitemap->add(URL::to('page'), '2012-08-26T12:30:00+02:00', '0.9', 'monthly');

        // get all posts from db, with image relations
        $posts = \DB::table('posts')->with('images')->orderBy('created_at', 'desc')->get();

        // add every post to the sitemap
        foreach ($posts as $post) {
            // get all images for the current post
            $images = [];
            foreach ($post->images as $image) {
                $images[] = [
                    'url' => $image->url,
                    'title' => $image->title,
                    'caption' => $image->caption
                ];
            }

            $sitemap->add($post->slug, $post->modified, $post->priority, $post->freq, $images);
        }
    }

    // show your sitemap (options: 'xml' (default), 'html', 'txt', 'ror-rss', 'ror-rdf')
    return $sitemap->render('xml');
});
```

### How to generate BIG sitemaps (with more than 1M items)

```php
Route::get('BigSitemap', function() {
    // create new sitemap object
    $sitemap = App::make('sitemap');

    // get all products from db (or wherever you store them)
    $products = DB::table('products')->orderBy('created_at', 'desc')->get();

    // counters
    $counter = 0;
    $sitemapCounter = 0;

    // add every product to multiple sitemaps with one sitemap index
    foreach ($products as $p) {
        if ($counter == 50000) {
            // generate new sitemap file
            $sitemap->store('xml', 'sitemap-' . $sitemapCounter);
            // add the file to the sitemaps array
            $sitemap->addSitemap(secure_url('sitemap-' . $sitemapCounter . '.xml'));
            // reset items array (clear memory)
            $sitemap->model->resetItems();
            // reset the counter
            $counter = 0;
            // count generated sitemap
            $sitemapCounter++;
        }

        // add product to items array
        $sitemap->add($p->slug, $p->modified, $p->priority, $p->freq);
        // count number of elements
        $counter++;
    }

    // you need to check for unused items
    if (!empty($sitemap->model->getItems())) {
        // generate sitemap with last items
        $sitemap->store('xml', 'sitemap-' . $sitemapCounter);
        // add sitemap to sitemaps array
        $sitemap->addSitemap(secure_url('sitemap-' . $sitemapCounter . '.xml'));
        // reset items array
        $sitemap->model->resetItems();
    }

    // generate new sitemapindex that will contain all generated sitemaps above
    $sitemap->store('sitemapindex', 'sitemap');
});
```

### How to generate sitemap to a file

```php
Route::get('mysitemap', function() {

    // create new sitemap object
    $sitemap = App::make("sitemap");

    // add items to the sitemap (url, date, priority, freq)
    $sitemap->add(URL::to(), '2012-08-25T20:10:00+02:00', '1.0', 'daily');
    $sitemap->add(URL::to('page'), '2012-08-26T12:30:00+02:00', '0.9', 'monthly');

    // get all posts from db
    $posts = DB::table('posts')->orderBy('created_at', 'desc')->get();

    // add every post to the sitemap
    foreach ($posts as $post) {
        $sitemap->add($post->slug, $post->modified, $post->priority, $post->freq);
    }

    // generate your sitemap (format, filename)
    $sitemap->store('xml', 'mysitemap');
    // this will generate file mysitemap.xml to your public folder

});

```

### How to use multiple sitemaps with sitemap index

```php
// sitemap with posts
Route::get('sitemap-posts', function() {
    // create sitemap
    $sitemap_posts = App::make("sitemap");

    // set cache
    $sitemap_posts->setCache('laravel.sitemap-posts', 3600);

    // add items
    $posts = DB::table('posts')->orderBy('created_at', 'desc')->get();

    foreach ($posts as $post) {
        $sitemap_posts->add($post->slug, $post->modified, $post->priority, $post->freq);
    }

    // show sitemap
    return $sitemap_posts->render('xml');
});
```

```php
// sitemap with tags
Route::get('sitemap-tags', function() {
    // create sitemap
    $sitemap_tags = App::make("sitemap");

    // set cache
    $sitemap_tags->setCache('laravel.sitemap-tags', 3600);

    // add items
    $tags = DB::table('tags')->get();

    foreach ($tags as $tag) {
        $sitemap_tags->add($tag->slug, null, '0.5', 'weekly');
    }

    // show sitemap
    return $sitemap_tags->render('xml');
});
```

```php
// sitemap index
Route::get('sitemap', function() {
    // create sitemap
    $sitemap = App::make("sitemap");

    // set cache
    $sitemap->setCache('laravel.sitemap-index', 3600);

    // add sitemaps (loc, lastmod (optional))
    $sitemap->addSitemap(URL::to('sitemap-posts'));
    $sitemap->addSitemap(URL::to('sitemap-tags'));

    // show sitemap
    return $sitemap->render('sitemapindex');
});
```

How to use multiple sitemaps with sitemap index (with store() method)

```php
Route::get('sitemap-store', function() {
    // create sitemap
    $sitemap_posts = App::make("sitemap");

    // add items
    $posts = DB::table('posts')->orderBy('created_at', 'desc')->get();

    foreach ($posts as $post) {
        $sitemap_posts->add($post->slug, $post->modified, $post->priority, $post->freq);
    }

    // create file sitemap-posts.xml in your public folder (format, filename)
    $sitemap_posts->store('xml','sitemap-posts');

    // create sitemap
    $sitemap_tags = App::make("sitemap");

    // add items
    $tags = DB::table('tags')->get();

    foreach ($tags as $tag) {
        $sitemap_tags->add($tag->slug, null, '0.5', 'weekly');
    }

    // create file sitemap-tags.xml in your public folder (format, filename)
    $sitemap_tags->store('xml','sitemap-tags');

    // create sitemap index
    $sitemap = App::make("sitemap");

    // add sitemaps (loc, lastmod (optional))
    $sitemap->addSitemap(URL::to('sitemap-posts'));
    $sitemap->addSitemap(URL::to('sitemap-tags'));

    // create file sitemap.xml in your public folder (format, filename)
    $sitemap->store('sitemapindex','sitemap');
});
```

## License

This package is open-sourced software licensed under the [MIT license](https://opensource.org/licenses/MIT).
