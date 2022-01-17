## Understanding lockForUpdate and sharedLock in Laravel


Recently, I stumbled upon the need for a mutex in my database for a critical use case — a game that allowed players to store inventory and wear equipment.

One thing that can go wrong is to have database problems and let users, by accident, for example, take off the same item twice and give them 2 items back instead of one in their inventory. Here’s how Laravel helped me tackle this problem without too much hassle. 🐱‍👤

<hr>

Transactional databases are fast and reliable, but not all the time. When you want to have high transaction numbers, you sacrifice something important for speed. That something important is atomicity. And here’s a real-world example that’s way more straightforward than a game — a banking system.

Let’s consider we have a very poor banking system in the pre-crypto era where our customers can easily transact between them. We are not really experienced with ledgers, so we choose to go with MySQL or Postgres (yes, even Postgres might go back if we don’t use locking).

Storing users’ details and the current balance is easy, right?

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class User extends Model
{
    protected $fillable = [
        'name',
        'balance',
    ];
    protected $casts = [
        'balance' => 'float',
    ];
}
```

Our bank is going to be the intermediary, so we can act as a class too:

```php
class Bank
{
    public static function sendMoney(User $from, User $to, float $amount)
    {
        if ($from->balance < $amount) {
            return false;
        }

        $from->update(['balance' => $from->balance - $amount]);
        $to->update(['balance' => $to->balance + $amount]);

        return true;
    }
}
```

And we can send money between the accounts:

```php
$alice = User::find(1); // 'balance' => 100
$bob = User::find(2); // 'balance' => 0

Bank::sendMoney($alice, $bob, 100); // true
```

You may ask now — okay, what’s wrong with it? It seems to do what it is supposed to, right?

Well, not quite. There are two major implementation issues that need to be addressed unless we want to have a double-spending issue with this centralized system.

<hr>

## 🛫 Database Transactions

The first major issue consists of the two lines where we update the balance of both customers.

```php
$from->update(['balance' => $from->balance - $amount]);
$to->update(['balance' => $to->balance + $amount]);
```

The major issue is that this is not a single point of failure. Meaning that if the first statement commits and the second one doesn’t, we will just **magically** erase some money from our system. We will subtract the money, but not add the money to Bob’s account.

To fix this, we will run [atomic transactions](https://laravel.com/docs/8.x/database#database-transactions):

```php
use Illuminate\Support\Facades\DB;

class Bank
{
    public static function sendMoney(User $from, User $to, float $amount)
    {
        if ($from->balance < $amount) {
            return false;
        }

        return DB::transaction(function () {
            $from->update(['balance' => $from->balance - $amount]);
            $to->update(['balance' => $to->balance + $amount]);
            return true;
        });
    }
}
```

In this particular case, the transaction will run, and in case it encounters any exception, it will roll back every single statement from before.

If the second transaction fails, it will roll back the first one, to make sure that Alice gets her money back, the money that was not able to get into Bob’s account.

<hr>

## 😥 Pessimistic Locking

We fixed the flaw of magically altering the inflation levels, but we face another issue: what if the balance changes mid-transaction? To better show the issue, we will have a new person that also needs money from Alice. Meet Charlie!

In this specific example, because we have a web app with thousands of HTTP web requests (slightly exaggerated, but banks do encounter this), we will ignore the fact that PHP is a blocking-IO programming language in the examples.

Let’s say that Alice has a fast sleight of hand, and both transactions where Alice sends money to Bob and Charlie are done, like at the same time, in the matter of microseconds. This means that if the odds are just right, the database will pull the records at the very same time, and send money at the very exact time.

If this scenario occurs, you will merely become stunned how from $100, you turned a total of $200 in the bank, $100 for each account.

The issue here is that we don’t have a locking mechanism in place for our database.

```php
// FIRST REQUEST
$alice = User::find(1); // 'balance' => 100,
$bob = User::find(2); // 'balance' => 0,
Bank::sendMoney($alice, $bob, 100); // true

// SECOND REQUEST
$alice = User::find(1); // 'balance' => 100,
$charlie = User::find(3); // 'balance' => 0,
Bank::sendMoney($alice, $charlie, 100); // true, but should have been false
```

This happens because if both queries run the SELECT statements (the ones defined by `find()`) at the same time, both requests will read that Alice has $100 in her account, which can be false because if the other transaction has already changed the balance, we remain with a reading saying she still has $100.

In this particular case, this is what might happen:

```
Request1: Reads Alice balance as $100
Request2: Reads Alice balance as $100
Request1: Subtract $100 from Alice
Request2: Subtract $100 from Alice
Request1: Add $100 to Bob
Request2: Add $100 to Charlie
```

The ideal situation would be this one:

```
Request1: Reads Alice balance as $100
Request1: Subtract $100 from Alice.
Request1: Add $100 to Bob
Request2: Reads Alice balance as $0
Request2: Don't allow Alice to send money
```

## ✅ Implementing Pessimistic Locking

Laravel has a neat way to tackle this issue with the help of queries. Databases (like MySQL) have a thing called [deadlock](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html). Deadlocks permit the one who runs a query to specifically describe whose rows can be selected or updated within a specific query.

Laravel has its own [documentation section about deadlocks](https://laravel.com/docs/8.x/queries#pessimistic-locking), but it was hard to digest which does what, so we have this awesome banking example.

Laravel documentation says:

> A “for update” lock prevents the selected records from being modified or from being selected with another shared lock.

This is what we want. If we run `lockForUpdate` in our `find()` statements, they will not be selected by another shared lock.

And for the shared lock:

> A shared lock prevents the selected rows from being modified until your transaction is committed.

Is this also what we want? Of course, if we apply this to the `find()` queries, the rows (in the first one Alice & Bob, in the second one Alice & Charlie) will not be read, nor modified until our `update` transaction got committed successfully.

```php
// FIRST REQUEST
DB::transaction(function () {
    $alice = User::lockForUpdate()->find(1); // 'balance' => 100
    $bob = User::lockForUpdate()->find(2); // 'balance' => 0

    Bank::sendMoney($alice, $bob, 100); // true
});

// SECOND REQUEST
DB::transaction(function () {
    $alice = User::lockForUpdate()->find(1); // 'balance' => 0
    $charlie = User::lockForUpdate()->find(3); // 'balance' => 0

    Bank::sendMoney($alice, $charlie, 100); // false
});
```

Obviously, having a `lockForUpdate` would be just enough, because, by definition, any rows selected by it will never be selected by another shared lock, either `lockForUpdate()` or `sharedLock()`.

Alternatively, just like the official Laravel writes, you may use `sharedLock()` just so other queries won’t select the same rows until the transaction is finished. The use case would be for strong read consistency, making sure that if another transaction may be in process, to not get outdated rows.

Thanks to Laravel and deadlocks, we can now avoid any inflation.👏

But if you decide to run your bank in Laravel, you should use [Event Sourcing](https://spatie.be/docs/laravel-event-sourcing/v5/introduction), you definitely don’t want to play with the market. 🤨

## 💸 Sponsorship

Hi, I'm [Alex](https://github.com/rennokki), the founder of [Renoki Co.](https://github.com/renoki-co). I'm thankful for taking your time to read this article, and I hope that it helped you. Developing and maintaining packages and delivering good articles about Laravel, Kubernetes and AWS takes a lot of time, but I believe it's a time well spent.

If you support more helpful articles, or you are using one or more Renoki Co. open-source packages in your production apps, in presentation demos, hobby projects, school projects or so, sponsor our work with [Github Sponsors](https://github.com/sponsors/rennokki). 📦