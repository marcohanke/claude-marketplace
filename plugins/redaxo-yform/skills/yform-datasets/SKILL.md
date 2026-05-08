---
name: yform-datasets
description: ORM-style read/write on YForm tables using rex_yform_manager_dataset and the query builder (rex_yform_manager_query). Covers model classes, all where/join/select/order/group methods, relations, pagination via rex_pager, CRUD, dataset-driven forms, and SQL debugging. Use when the user reads or writes to a YForm table from PHP, builds custom backend lists, joins related tables, paginates, or queries datasets in frontend templates.
---

# YForm Datasets (YOrm)

`rex_yform_manager_dataset` is YForm's active-record-style ORM. One subclass per table, automatic loading/saving, lazy relations, fluent query builder.

## Two ways to use it

### Without a model class

```php
$items = rex_yform_manager_table::get('rex_my_table')->query()->find();
```

### With a model class (recommended)

```php
// lib/MyTable.php
class MyTable extends rex_yform_manager_dataset {}

// boot.php
rex_yform_manager_dataset::setModelClass('rex_my_table', MyTable::class);

// Anywhere
$items = MyTable::query()->find();
```

After `setModelClass()`, queries on the table return your subclass instances.

### Custom model with methods

```php
class team_member extends rex_yform_manager_dataset {
    public function getName(): string {
        return (string) $this->getValue('name');
    }

    public function getPhotoMedia(): ?rex_media {
        $filename = (string) $this->getValue('photo');
        return $filename ? rex_media::get($filename) : null;
    }

    public function getRole(): ?team_role {
        $relation = $this->getRelatedDataset('role');
        return $relation instanceof team_role ? $relation : null;
    }
}
```

## Query builder

```php
$query = MyTable::query();
$query
    ->alias('t')
    ->joinRelation('category_id', 'c')
    ->select('c.name', 'category_name')
    ->where('t.status', '1')
    ->where('t.created', $date, '>')
    ->orderBy('t.name')
    ->limit(0, 10);

$items = $query->find();
```

### All query methods

**Where:** `where`, `whereNot`, `whereNull`, `whereNotNull`, `whereBetween`, `whereNotBetween`, `whereNested`, `whereRaw`, `whereListContains`, `setWhereOperator`, `resetWhere`
**Having:** `having`, `havingNot`, `havingNull`, `havingNotNull`, `havingBetween`, `havingNotBetween`, `havingRaw`, `havingListContains`, `setHavingOperator`, `resetHaving`
**Join:** `joinRelation`, `leftJoinRelation`, `joinRaw`, `joinType`, `joinTypeRelation`, `leftJoin`, `resetJoins`
**Select:** `select`, `selectRaw`, `resetSelect`
**Order:** `orderBy`, `orderByRaw`, `resetOrderBy`
**Group:** `groupBy`, `groupByRaw`, `resetGroupBy`
**Other:** `alias`, `limit`, `resetLimit`, `count`, `exists`, `find`, `findOne`, `findId`, `findIds`, `paginate`, `save`, `getQuery`, `getParams`

### WHERE patterns

```php
// Equality / comparison
->where('status', 1)
->where('age', 18, '>=')
->where('email', '%@example.com', 'LIKE')

// IN clause
->where('status', [1, 2])

// Raw SQL (for things the builder can't express)
->whereRaw('LOWER(name) = LOWER(?)', ['Alice'])

// List contains (for comma-separated values stored in a column)
->whereListContains('tags', 5)
->whereListContains('tags', [3, 5, 9])

// OR conditions
$query->setWhereOperator('OR')
    ->where('foo', 1)
    ->where('bar', 2);

$query->whereRaw('(foo = :foo OR bar = :bar)', ['foo' => 1, 'bar' => 2]);

// OR with whereNested (array form)
$query->whereNested(['foo' => 1, 'bar' => 2], 'OR');

// OR with whereNested (callback form)
$query->whereNested(function (rex_yform_manager_query $query) {
    $query->where('foo', 1)->where('bar', 2);
}, 'OR');
```

### Pagination

```php
$pager = new rex_pager(20); // 20 items per page
$items = MyTable::query()->where('status', 1)->paginate($pager);

$fragment = new rex_fragment();
$fragment->setVar('urlprovider', rex_article::getCurrent());
$fragment->setVar('pager', $pager);
echo $fragment->parse('core/navigations/pagination.php');

foreach ($items as $item) {
    echo $item->getValue('name');
}

// Pager info
$pager->getRowCount();
$pager->getCurrentPage();
$pager->getLastPage();
$pager->getPageCount();
```

### Single record

```php
$member = MyTable::get(42);              // by primary key – nullable
$member = MyTable::query()->where('email', $email)->findOne();
```

## CRUD

```php
// Create
$item = rex_yform_manager_dataset::create('rex_my_table');
// or with model class: $item = MyTable::create();
$item->setValue('name', 'Test');
$item->save();

// Read
$item = rex_yform_manager_dataset::get($id, 'rex_my_table');
$item = MyTable::get($id);

// Update
$item = MyTable::get($id);
$item->name = 'New Name';
$item->save();

// Delete
$item = MyTable::get($id);
$item->delete();

// Bulk delete via collection
MyTable::query()->where('status', 0)->find()->delete();
```

`save()` runs the table's configured validators automatically. Failures populate `getMessages()`. Always check the return value or log messages.

## Relations

```php
// "to-one" (e.g. author of a post)
$author = $post->getRelatedDataset('author_id'); // dataset or null

// "to-many" (e.g. all posts of an author)
$posts = $author->getRelatedCollection('posts'); // collection

// Join in a query
$query = MyTable::query()
    ->joinRelation('author_id', 'a')
    ->selectRaw('CONCAT(a.first_name, " ", a.last_name)', 'author_name');
```

The string passed to `getRelatedDataset()` matches the field `name` on the originating table.

## Forms from a dataset

### Edit existing

```php
$dataset = MyTable::get($id);
$yform = $dataset->getForm();
$yform->setObjectparams('form_action', rex_getUrl(REX_ARTICLE_ID));
$yform->setActionField('showtext', ['', 'Saved']);
echo $dataset->executeForm($yform);
```

### Create new

```php
$dataset = MyTable::create();
$yform = $dataset->getForm();
$yform->setObjectparams('form_action', rex_getUrl(REX_ARTICLE_ID));
$yform->setActionField('showtext', ['', 'Saved']);
echo $dataset->executeForm($yform);
```

## Collections

`rex_yform_manager_collection` (the result of `find()`) has these methods:

`delete`, `filter`, `first`, `last`, `getIds`, `getValues`, `groupBy`, `isEmpty`, `map`, `save`, `setValue`, `shuffle`, `slice`, `sort`, `split`, `toKeyIndex`, `toKeyValue`, `isValueUnique`, `getUniqueValue`, `populateRelation`

## Filter fields for the frontend

When a dataset is used in both backend (admin) and frontend (public) contexts, hide internal fields from frontend forms:

```php
class MyDataset extends rex_yform_manager_dataset {
    public function getFields(array $filter = []) {
        $fields = $this->getTable()->getFields($filter);
        if (rex::isBackend()) return $fields;
        foreach ($fields as $i => $field) {
            if (in_array($field->getName(), ['internal_links', 'admin_notes'])) {
                unset($fields[$i]);
            }
        }
        return $fields;
    }
}
```

## Frontend usage example

```php
<?php
$members = team_member::query()
    ->where('status', 1)
    ->orderBy('prio')
    ->find();
?>
<ul class="team">
<?php foreach ($members as $m): ?>
    <li>
        <h3><?= rex_escape($m->getName()) ?></h3>
        <?php if ($photo = $m->getPhotoMedia()): ?>
            <img src="<?= rex_url::media($photo->getFileName()) ?>"
                 alt="<?= rex_escape($photo->getTitle()) ?>"
                 width="<?= $photo->getWidth() ?>"
                 height="<?= $photo->getHeight() ?>">
        <?php endif; ?>
        <?php if ($role = $m->getRole()): ?>
            <p class="role"><?= rex_escape($role->getName()) ?></p>
        <?php endif; ?>
    </li>
<?php endforeach; ?>
</ul>
```

## Backend list extensions

Hide a column from the Table Manager list view:

```php
rex_extension::register('YFORM_DATA_LIST', function (rex_extension_point $ep) {
    $list = $ep->getSubject();
    $list->removeColumn('internal_field');
});
```

Custom column rendering:

```php
rex_extension::register('YFORM_DATA_LIST', function (rex_extension_point $ep) {
    $list = $ep->getSubject();
    $list->setColumnFormat('image', 'custom', function ($params) {
        $value = $params['list']->getValue('image');
        if ($value) {
            return '<img src="/media/' . rex_escape($value) . '" style="max-width:80px;">';
        }
        return '';
    });
});
```

## CSRF tokens for Table Manager links

When you build custom links into the Table Manager backend (e.g. an "Edit" button from a frontend page that opens the backend), add the CSRF token:

```php
$_csrf_key    = $table->getCSRFKey();
$_csrf_params = rex_csrf_token::factory($_csrf_key)->getUrlParams();
// Append $_csrf_params to your URL
```

## Debugging

```php
// Method 1: dump the SQL + params for a query
$query = MyTable::query()->where('status', 1);
rex_sql::factory()->setDebug()->getArray($query->getQuery(), $query->getParams());

// Method 2: global debug flag in dataset.php (set $debug = true)
```

## Quick reference: core classes

| Class | Purpose |
|---|---|
| `rex_yform_manager_table` | Table metadata & form generation |
| `rex_yform_manager_dataset` | ORM entity (CRUD) |
| `rex_yform_manager_query` | Query builder |
| `rex_yform_manager_collection` | Dataset collection |

## Common pitfalls

- Querying without `setModelClass()` registered → returns plain `rex_yform_manager_dataset` instead of your subclass; method calls fail.
- Calling `getValue('photo')` and treating the result as a `rex_media` instance – it's a filename string. Look up `rex_media::get()` separately.
- Forgetting `save()` returns `false` on validation failure. Always check the return or log `getMessages()`.
- Loading thousands of rows with `find()` then iterating – use `findIds()` and process in batches, or write a raw `rex_sql` query.
- Using `whereListContains` on a column that isn't a comma-separated list – matches accidentally on substrings.
- `paginate($pager)` without setting up `rex_pager` first with the page size – returns the default 20, may not match expectations.
- Forgetting `->alias('t')` when using `joinRelation` – causes "ambiguous column" errors when the joined table has same-named columns.
