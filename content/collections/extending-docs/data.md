---
id: 0267eb5a-f54b-4e3f-bef0-1f16762720b1
title: Data Retrieval and Manipulation
intro: One of the most crucial aspects of extending a Content Management System is being able to retrieve the data and manipulate it. Statamic has a number of classes to provide you with ways to handle these sorts of situations.
---

Consider the various aspects of Statamic: Entries, Terms, Globals, and Assets. They are all Data. Data can have variables/fields that you can get, set, etc.

## Facade Primer

In most cases, the first point of contact with Statamic functionality will be through a Facade.

You can find more details on which ones to use later, but you will find them all in the `Statamic\Facades` namespace. Of course there are exceptions, but in most cases you will be looking for a Facade.

Each facade will proxy method calls to another class. You can see which class by looking for the `getFacadeAccessor` method.

Some will be simple, direct class mappings, like the [YAML facade](https://github.com/statamic/cms/blob/master/src/Facades/YAML.php#L17-L20).

``` php
Statamic\Facades\YAML::parse();
// This calls the `parse` method on an instance of `Statamic\Yaml\Yaml`
```

Some reference a contract, which could change depending on how an application is configured, like the [Entry facade](https://github.com/statamic/cms/blob/master/src/Facades/Entry.php#L27-L30). This class references the `EntryRepository` contract, which by default is bound to the Stache implementation, but could be changed to use databases, etc.

``` php
Statamic\Facades\Entry::make();
// This calls the `make` method on an instance of `Statamic\Contracts\Entries\EntryRepository`
// By default it's `Statamic\Stache\Repositories\EntryRepository`, but could change.
```

The facades will have a `@see` annotation in their docblock to give you a hint on where to look.


## Retrieving Data

You should retrieve data using Facade methods. If you’ve used Laravel, it should feel similar to Eloquent. If it helps, try thinking of each data type mentioned above as a Model. We have a Facade for each of those.

For example, this will find an entry with an ID of `f6d5a87`.

``` php
$entry = \Statamic\Facades\Entry::find('f6d5a87');
```

Each data type may have more methods for retrieving data. You can also find an entry by it's slug or URI:

``` php
Entry::findBySlug('shoes', 'clothing');
Entry::findByUri('/clothing/shoes');
Entry::findByUri('/vetements/chaussures', 'french'); // For multisite
```

Like Laravel, if you’re expecting a collection of models, you will receive a collection. However, Statamic will give you a subclass like `EntryCollection` which will do everything `Illuminate\Support\Collection` does [(docs)](https://laravel.com/docs/collections), with a few more contextual methods at your disposal should you need them.

If you’re expecting a single model you’ll get the corresponding class. (In the example above, you'll get a `Statamic\Entries\Entry` instance).

Once you have your objects, you may get data out of them in [a handful of ways](#methods).


## Manipulating Data

Once you have a data instance, you can go to town on it.

``` php
$entry->set('foo', 'bar');
```

This is like adding `foo: bar` to the front-matter of the entry file.

Once you’re done, go ahead and save it.

``` php
$entry->save();
```

Now it’ll be written to file. Nice.

### Events
When you are saving or creating your data instance, the `EntrySaving`, `EntryCreated` and the `EntrySaved` events are dispatched.  In some cases, you would rather suppress those events. For example, to prevent causing an infinite loop of `EntrySaved` events.
``` php
$entry->saveQuietly();
```

## Creating Data

Of course, the data had to get there somehow. You can also create data using the corresponding facades.

Each of them has a `make` method that will give you a new instance.
Once you have an instance, you can manipulate it using various methods the same way as if it already existed. Most of the time, these are chainable to give you a nice fluent interface:

``` php
use Statamic\Facades\Entry;

$entry = Entry::make()
            ->published()
            ->data(['title' => 'About us', 'subtitle' => 'We are awesome'])
            ->etc(); // and so on...

$entry->save();
```

:::tip
Make sure to use the `make` method, rather than simply `new`'ing up a class. For example, if a user has customized their application to store entries in a database, they will have a different Entry class. Using `Entry::make()` will make sure to get the correct class.
:::

## Getting Field Data

### Data
This would be the data defined directly on the item, like what you'd find in an entry's YAML front-matter. (Some specific keys may be stripped out, like an entry's `id`).

```yaml
id: 123
title: My post
content: The post content
```

```php
$entry->data();
// Illuminate\Support\Collection([
//    'title' => 'My post',
//    'content' => 'The post content'
// ])
```

You can use the `get` method to get a single field's data.

```php
$entry->get('title') // 'My post'
```

More often than not, you'll want to be using `values` or `value` instead.

### Values
The "values" are similar to data, except they will also inherit from any originating items. For example, if an entry has been localized
from another entry.

```yaml
id: 123
title: My post
content: The post content
```

```yaml
id: 456
origin: 123
title: My localized post
```

```php
$entry->values();
// Illuminate\Support\Collection([
//    'title' => 'My localized post',
//    'content' => 'The post content'
// ])
```

You can use the `value` method to get a single field's value.

```php
$entry->value('title'); // 'My localized post'
```

### Augmented Values

Items that support augmentation (e.g. entries) will be able to provide an augmented version of themselves.

Read more about [Augmentation](/extending/augmentation).

You can get a single augmented value:

```php
$entry->augmentedValue('title'); // Statamic\Fields\Value
```

All the augmented values:

``` php
$entry->toAugmentedArray();
// [
//    'title' => Statamic\Fields\Value,
//    'content' => Statamic\Fields\Value,
//    'collection' => Statamic\Entries\Collection,
//    'uri' => '/posts/my-post',
//    ...etc...
// ]
```

Or a subset of augmented values:

```php
$entry->toAugmentedArray(['title', 'collection']);
// [
//    'title' => Statamic\Fields\Value,
//    'collection' => Statamic\Entries\Collection,
// ]
```
