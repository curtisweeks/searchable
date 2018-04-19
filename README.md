Laravel Searchable
==========================================

Laravel 5 trait to add a `search` method for Eloquent with priorities for each searchable field for the model.

## Installation

```
$ composer require "jabbtech/searchable": "~2.0"
```

## Configuration

Import the `SearchableTrait` class and add a `$searchable` property to your model to specify the columns to make searchable and their priority in search results. Columns with higher values are more important:

```php
namespace App;

use Illuminate\Database\Eloquent\Model;
use Jabbtech\Searchable\SearchableTrait;

class User extends Model
{

    use SearchableTrait;

    protected $searchable = [
        'columns' => [
            'users.first_name' => 10,
            'users.last_name' => 10,
            'users.bio' => 2,
            'users.email' => 5,
            'posts.title' => 2,
            'posts.body' => 1,
        ],
        'joins' => [
            'posts' => ['users.id','posts.user_id'],
        ],
    ];

    public function posts()
    {
        return $this->hasMany('App\Post');
    }

}
```

## Basic Usage

Using the `search` method on a query builder instance builds a query that searches through your model using Laravel's Eloquent. The most basic call to `search` requires a single argument, a search string.

```php
$users = User::search($query)->get();
```

The `search` method should be compatible with any eloquent method (with exceptions for [SQL Server](#sql-server)):
```php
$users = User::search($query)->with('posts')->get();

$users = User::search($query)->with('posts')->paginate(20);

$users = User::search($query)->where('status', 'active')->take(10)->get();
```

#### Accepted Relevance Threshold

The second argument changes is an integer that can change the default threshold for accepted relevance. By default, the threshold for accepted relevance is the sum of all attribute relevance divided by 4.

Returning all users matching the search in order of relevance:

```php
$users = User::search($query, 0)->get();
```

#### Prioritize Full Text Search

The third argument is a boolean which prioritizes matches that include the multi-word search. By default, multi-word search terms are split and Searchable searches for each word individually. Relevance plays a role in prioritizing matches that matched on multiple words.

Prioritizing matches containing "John Doe" above matches containing only "John" or "Doe":

```php
$users = User::search("John Doe", null, true)->get();
```

#### Full Text Matches Only

The fourth argument is a boolean which allows searching for full text mathces only.

Excluding matches that only matched "John" OR "Doe".

```php
$users = User::search("John Doe", null, true, true)->get();
```

---

## SQL Server
When used with SQL Server the `search` method supports most eloquent methods with some limitations.
* It is strongly recommended (and required in some cases) that the `select` method is used
    * Column constraints of tables you will join to need to be included in the `select` method
    * Column required by the `where` method need to be included in the `select` method
* Some methods need to be included _before_ the `search` method
    * `join`
    * `leftJoin`
    * `crossJoin`
    * `where`
    * `take`
* Include `orderBy('relevance', 'desc')` _after_ the `search` method

```php
$users = User::select('first_name', 'last_name')
    ->search($query)
    ->orderBy('relevance', 'desc')
    ->get();

$users = User::select(
        'users.id',
        'users.first_name',
        'users.last_name',
        'users.bio',
        'users.email',
        'users.status',
        'posts.title',
        'posts.body'
    )->leftJoin('posts', 'users.id', 'posts.user_id')
    ->where('users.status', 'active')
    ->take(50)
    ->serch($query)
    ->orderBy('relevance', 'desc')
    ->get();
```

---

## How does it work?

Searchable builds a query that searches through your model using Laravel's Eloquent. Here is an example query

#### Model
```php
protected $searchable = [
    'columns' => [
        'first_name' => 10,
        'last_name' => 5
    ],
];
```

#### Search
```php
$search = User::search('John Doe', null, true)->get();
```

#### Result
```sql
select * from (
    select `users`.*, max(
        (case when LOWER(`users`.`first_name`) LIKE 'John' then 150 else 0 end) + -- column equals word: priority * 15
        (case when LOWER(`users`.`first_name`) LIKE 'Doe' then 150 else 0 end) +
        (case when LOWER(`users`.`first_name`) LIKE 'John%' then 50 else 0 end) + -- column starts with word: priority * 5
        (case when LOWER(`users`.`first_name`) LIKE 'Doe%' then 50 else 0 end) +
        (case when LOWER(`users`.`first_name`) LIKE '%John%' then 10 else 0 end) + -- column contains word: priority * 1
        (case when LOWER(`users`.`first_name`) LIKE '%Doe%' then 10 else 0 end) +
        (case when LOWER(`users`.`first_name`) LIKE 'John Doe' then 500 else 0 end) + -- column matches full text: priority * 50
        (case when LOWER(`users`.`first_name`) LIKE '%John Doe%' then 300 else 0 end) + -- column contains full text: priority * 30
        (case when LOWER(`users`.`last_name`) LIKE 'John' then 75 else 0 end) + 
        (case when LOWER(`users`.`last_name`) LIKE 'Doe' then 75 else 0 end) + 
        (case when LOWER(`users`.`last_name`) LIKE 'John%' then 25 else 0 end) + 
        (case when LOWER(`users`.`last_name`) LIKE 'Doe%' then 25 else 0 end) + 
        (case when LOWER(`users`.`last_name`) LIKE '%John%' then 5 else 0 end) + 
        (case when LOWER(`users`.`last_name`) LIKE '%Doe%' then 5 else 0 end) + 
        (case when LOWER(`users`.`last_name`) LIKE 'John Doe' then 250 else 0 end) + 
        (case when LOWER(`users`.`last_name`) LIKE '%John Doe%' then 150 else 0 end)
    ) as relevance 
    from `users` 
    group by `users`.`id` 
    having relevance >= 3.75 -- sum of priorities (10 + 5) / 4
    order by `relevance` desc
) as `users`
```

## Credits

* [nicolaslopezj/searchable](https://github.com/nicolaslopezj/searchable)

## Contributing

Anyone is welcome to contribute. Fork, make your changes, and then submit a pull request.