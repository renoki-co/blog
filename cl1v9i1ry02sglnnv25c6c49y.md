## Why Laravel is the best framework to start and learn if you're trying to build production-ready apps

Just a few days ago I've started working with a new client, whose framework consisted of Zend Framework.

%[https://twitter.com/rennokki/status/1511956399198244864?s=20&t=FbjTFzAETHlS6pvjwDsa5g]

As a Certified Laravel developer with 5+ years of experience of production-grade apps, when I saw their source code, I asked "why did i kept working with Laravel during all these years?".

At glance, I didn't know why. It was more of a [*je ne sais quoi*](https://www.merriam-webster.com/dictionary/je%20ne%20sais%20quoi). So in the need of seeking answers, I've put up a consistent list of why to pick Laravel as your next framework, no matter the size of your project.

## Simple Installation

If you need to scaffold a project, you don't need to be copying files or following complex installation processes that require time to read and understand.

You can have a running server like this:

```bash
composer create-project laravel/laravel blog
```

```bash
cd blog
```

```bash
php artisan serve

Starting Laravel development server: http://127.0.0.1:8000        
[Sun Apr 10 18:54:17 2022] PHP 8.1.2 Development Server (http://127.0.0.1:8000) started
```

## Boilerplating

Tired of implementing authentication again and again, for each project? Laravel already has boilerplated a lot of things, coming as [Starter Kits](https://laravel.com/docs/9.x/starter-kits) from simple authentication pages with Laravel Breeze to complex boilerplates with Vue.js and full-dashboard applications, with Two-Factor Authentication.

Basically, you have zero reimplementation of business logic that's critical to your next app.

This also applies to first-party and third-party packages, like [Laravel Cashier](https://laravel.com/docs/master/billing), where you can charge your customers with Stripe or Paddle without writing any prior code. I will explain more later in the article.

## Database-table-as-a-class

Writing SQL can be easy, but it isn't fun when things become more complex. Having a 1-on-1 pair PHP class with a table seems like a neat choice. [Laravel ORM is wonderful](https://laravel.com/docs/master/eloquent), as you can create one for any table:

```bash
php artisan make:model Post -f
```

The `-f` flag is from `F`actory and is telling Laravel to also create a factory for us, so that we can create posts for testing later.

```php
// app/Models/Post.php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    use HasFactory;
}
```

```php
$post = new Post;

$post->user_id = User::first();
$post->title = 'Laravel The Easy Way';
$post->body = 'This is the content.';
$post->save();
```

We can query a lot of things:

```php
$allPosts = Post::all();

$post = Post::where('title', 'Laravel The Easy Way')->first();

$latestPosts = Post::orderBy('created_at', 'desc')->limit(5)->get();
```

Another powerful feature of models is that you can create [relationships](https://laravel.com/docs/master/eloquent-relationships) between tables (or classes, in this case):

```php
// app/Models/Post.php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    use HasFactory;

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

Now we can get the posts with the user:

```php
foreach (Post::with('user')->get() as $post) {
    // $post->user->name is the User's name
}
```

## Database versioning

No SQL files to carry around, no GBs of dumps to receive from your colleagues to be able to develop locally. [With migrations, you all have the same database structure](https://laravel.com/docs/9.x/migrations).

```bash
php artisan make:migration create_posts_table
```

```php
// database/migrations/xxxx_xx_xx_000000_create_posts_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class() extends Migration {
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();
            $table->unsignedBigInteger('user_id')->index(); // <-- index for faster query ⚡
            $table->string('title');
            $table->text('body');
            $table->timestamps(); // <-- will create created_at and updated_at
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('posts');
    }
};
```

To create the table, one command:

```bash
php artisan migrate

Migrating: xxxx_xx_xx_000000_create_posts_table.php
Migrated: xxxx_xx_xx_000000_create_posts_table.php
```

If running again:

```bash
php artisan migrate

Nothing to migrate.
```

You can also pair it up with [some seeders](https://laravel.com/docs/9.x/seeding) so you can populate the database to have some testing data right after migration, so you won't have to create database users or posts by hand, each time.

Previously, we created the model with `-f`, so we can configure the way our records will be created for seeding:

```php
// database/factories/PostFactory.php

namespace Database\Factories;

use App\Models\Post;
use Illuminate\Database\Eloquent\Factories\Factory;

class PostFactory extends Factory
{
    /**
     * The name of the factory's corresponding model.
     *
     * @var string
     */
    protected $model = Post::class;

    /**
     * Define the model's default state.
     *
     * @return array
     */
    public function definition()
    {
        return [
            'user_id' => User::factory()->create(), // <-- creates one user
            'title' => $this->faker->sentence(),
            'body' => $this->faker->text(),
        ];
    }
}
```

Later, we can use the `::factory()` static method to create posts randomly:

```php
// database/seeders/DatabaseSeeder.php

namespace Database\Seeders;

use App\Models\Post;
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    /**
     * Seed the application's database.
     *
     * @return void
     */
    public function run()
    {
        Post::factory(10)->create(); // <-- create 10 posts
    }
}
```

You can read more about factories [here](https://laravel.com/docs/master/database-testing#defining-model-factories).

## Your MVC friend 😍

Stop complicating yourself with Admin UIs to build pages. One command and you have the [controller](https://laravel.com/docs/master/controllers):

```bash
php artisan make:controller BlogController
```

You can create something like this:

```php
namespace App\Http\Controllers;

use Illuminate\Http\Request;

class BlogController extends Controller
{
    /**
     * Show the homepage.
     *
     * @return \Illuminate\Http\Response
     */
    public function home()
    {
        return view('home', [
            'posts' => Post::with('user')->get(), // <-- inject all posts from the database as $posts
        ]);
    }

    /**
     * Create a post.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    public function createPost(Request $request)
    {
        $data = $request->validate([
            'title' => ['required', 'string', 'max:255'],
            'body' => ['required', 'string', 'max:1000'],
        ]);

        $post = new Post;

        $post->title = $data['title'];
        $post->body = $data['body'];
        $post->user_id = 1;

        $post->save();

        return view('home');
    }
}
```

The [routes](https://laravel.com/docs/master/routing) are defined as OOP:

```php
// routes/web.php

use App\Http\Controllers\BlogController;

Route::get('/', [BlogController::class, 'home']);
Route::post('/', [BlogController::class, 'createPost']);
```

For the views, you can use the default [Blade engine](https://laravel.com/docs/master/blade), that's really powerful for writing HTML without actually using raw PHP in the code:

```html
<html>
    <body>
        @foreach ($posts as $post)
            <div>
                "{{ $post->title }}" written by {{ $post->user->name }}: {{ $post->body }}
            </div>
        @endforeach
    </body>
</html>
```

## Queueing for background processing

Sometimes, you might want to process batches of tasks, concurrently, or defering them from the user request, like generating reports or sending mails.

[Laravel has a built-in queue system](https://laravel.com/docs/master/queues) that lets you define Jobs to process in the background. For this feature, to work in the background, you will need Redis. Otherwise, the jobs will by default run in the same process.

```php
php artisan make:job GeneratePostReport
```

This will create a Job file:

```php
namespace App\Jobs;

use App\Models\Post;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class GeneratePostReport implements ShouldQueue
{
    use Dispatchable;
    use InteractsWithQueue;
    use Queueable;
    use SerializesModels;

    /**
     * Create a new job instance.
     *
     * @param  \App\Models\Post  $post
     * @return void
     */
    public function __construct(protected Post $post)
    {
        //
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        $report = $this->post->generateReport();

        // Save the report or something...
    }
}
```

Using [Horizon](https://laravel.com/docs/master/horizon) and a localhost Redis server, you can start the workers:

```bash
composer require laravel/horizon
```

```bash
# this task has to be ran in background, as it blocks the I/O
php artisan horizon
```

Create a Job file from the CLI:

```bash
php artisan make:job GeneratePostReport
```

```php
// app/Jobs/GeneratePostReport.php

namespace App\Jobs;

use App\Models\Post;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class GeneratePostReport implements ShouldQueue
{
    use Dispatchable;
    use InteractsWithQueue;
    use Queueable;
    use SerializesModels;

    /**
     * Create a new job instance.
     *
     * @param  \App\Models\Post  $post
     * @return void
     */
    public function __construct(protected Post $post)
    {
        //
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        $report = $this->post->generateReport(); // <-- Save the report or something...
    }
}
```

In your code, either it's your controller, console or another job, you can dispatch the job to the queues:

```php
use App\Jobs\GeneratePostReport;
use App\Models\Post;

GeneratePostReport::dispatch(
    Post::where('title', 'Laravel The Easy Way')->first()
);
```

## Built-in mailing

Another queueing-tied feature can be sending mails. When was the last time you wrote a mail, programatically, to inform your subscribers about posts? Remember how hard it is?

[Laravel makes this simple](https://laravel.com/docs/master/mail) by programatically defining a mail without any design knowledge, by using markdown:

```bash
php artisan make:mail PostPublished --markdown=emails.posts.published
```

```php
namespace App\Mail;
 
use App\Models\Post;
use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Queue\SerializesModels;
 
class PostPublished extends Mailable
{
    use Queueable, SerializesModels;

    /**
     * Create a new message instance.
     *
     * @param  \App\Models\Post  $post
     * @return void
     */
    public function __construct(Post $post)
    {
        //
    }
 
    /**
     * Build the message.
     *
     * @return $this
     */
    public function build()
    {
        return $this->from('example@test.com')
            ->markdown('emails.posts.published', [
                'post' => $this->post->load('user'), // <-- Inject post with User as $post
            ]);
    }
}
```

```html
@component('mail::message')
# Post Published
 
A new post was published.

## {{ $post->title }} by {{ $post->user->name }}

{{ $post->body }}
@endcomponent
```

You can later send the mail to your subscribers:

```php
use App\Models\Post;
use App\Mail\PostPublished;
use Illuminate\Support\Facades\Mail;

$post = Post::where('title', 'Laravel The Easy Way')->first();

foreach (Subscribers::all() as $subscriber) {
    Mail::to($subscriber->email)->send(new PostPublished($post));
}
```

## Task Scheduling

One of the features you will want often tare time-based tasks, like generating reports and running code at arbitrary time. This can be achieved with a custom command, and run it in the [Task Scheduling](https://laravel.com/docs/master/scheduling):

```php
php artisan make:command SendLastDayPosts
```

```php
namespace App\Console\Commands;

use App\Models\Post;
use Illuminate\Console\Command;

class SendLastDayPosts extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'send:last-day-posts';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Send the posts from yesterday to your subscribers.';

    /**
     * Execute the console command.
     *
     * @return int
     */
    public function handle()
    {
        $posts = Post::whereBetween('created_at', [
            now()->subDay()->startOfDay(),
            now()->subDay()->endOfDay(),
        ])->get();

        foreach ($posts as $post) {
            // Send the mail or something...
        }

        return 0;
    }
}
```

The command can be registered as  call in the task scheduler:

```php
// app/Console/Kernel.php

namespace App\Console;

use Illuminate\Console\Scheduling\Schedule;
use Illuminate\Foundation\Console\Kernel as ConsoleKernel;

class Kernel extends ConsoleKernel
{
    /**
     * Define the application's command schedule.
     *
     * @param  \Illuminate\Console\Scheduling\Schedule  $schedule
     * @return void
     */
    protected function schedule(Schedule $schedule)
    {
        $schedule->command('send:last-day-posts')->dailyAt(3); // <-- every day, at 3 AM UTC (or server time)
    }

    /**
     * Register the commands for the application.
     *
     * @return void
     */
    protected function commands()
    {
        $this->load(__DIR__.'/Commands');

        require base_path('routes/console.php');
    }
}
```

Laravel will decide which tasks to run, just by calling the `schedule:run` command each minute, [in your crontab file](https://laravel.com/docs/master/scheduling#running-the-scheduler):

```
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```

## Great Ecosystem

Laravel is one of, if not the, best maintainer-supported and community-supported framework.

Laravel first-party packages, like [Passport](https://laravel.com/docs/master/passport) for OAuth login, [Socialite](https://laravel.com/docs/master/socialite) for very easy social media authentication, or even simple development environments with [Sail](https://laravel.com/docs/master/sail) for Docker, [Valet](https://laravel.com/docs/master/valet) for Mac, or [Homestead](https://laravel.com/docs/master/homestead) for VirtualBox.

Laravel has a great support to implement packages to publish on Packagist, that you can easily update across multiple projects, without having to re-write code all over again.

Alongside, the community does a great job at creating packages for Laravel, even since the early days of Laravel. [Spatie](https://github.com/spatie) and [BeyondCode](https://github.com/beyondcode) are two of the best community-based developers for Laravel packages that are covering a lot of use cases for your everyday projects.

You can keep in touch with Laravel's latest trends, packages and tutorials on their [Laravel News](https://laravel-news.com/category/packages) page.

Here, at [Renoki Co.](https://github.com/renoki-co), we try our best to help Laravel developers with well-written, maintained, free code for their projects.

## Bonus: Great speed for PHP

PHP can be sped up with OPCache, natively. However, [the bootstrapping process of Laravel packages is taking place every request](https://laravel.com/docs/master/lifecycle).

[The Laravel team maintains Octane](https://laravel.com/docs/master/octane), a package that allows your Laravel applications to be blazing fast, by [compiling the bootstrapping process in-memory and achieve great speeds](https://laravel-news.com/laravel-octane).

## Bonus: Enterprise uses Laravel

[When the Twitch leak got viral](https://arstechnica.com/information-technology/2021/10/twitch-admits-to-major-leak-exposing-source-code-creator-earnings/), one of the people I follow on Twitter shared a proof that internally - they used Laravel.

%[https://twitter.com/m1guelpf/status/1445713651403399171?s=20&t=dK4Y6qM88pD3fFXR4G0Slg]