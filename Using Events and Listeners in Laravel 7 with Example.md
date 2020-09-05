# Using Events and Listeners in Laravel 7 with Example

Laravel provides an excellent way to hook into a certain event in your application using **Events and Listeners**. This feature will allow you to subscribe and listen to various activities that took place in your application.

In simple words, think of **event** as something that has occurred in your application and **listeners** as a set of logic to respond with. Therefore, this allows us, developers, to make announcements within the application that something has happened and perform a set of operations based on that specific event.

As a result, **Events and Listeners** will further help to simplify our codes and refactor complicated tasks.

In this article, we’ll create **Events and Listeners** using an example of a **post**. Once a post is created, the users will be notified about the post through emails.

### Creating Events and Listeners

First, we’ll start by creating an event `PostCreated` and a listener `NotifyPostCreated`. You can use the following artisan command to create one.

```
php artisan make:event PostCreated
php artisan make:listener NotifyPostCreated --event="PostCreated"
```

The command above will create events and listeners inside directories the ‘**app/Events**’ and ‘**app/Listeners**‘ folders respectively.

After that, you need to register your events and listeners in `app/Providers/EventServiceProvider.php` file.

The protected `$listen` property contains an array of all events and their listeners as key-value pairs. Next, we add the event `PostCreated` and the listener `NotifyPostCreated` to array `$listen` as shown below.

```
use App/Events/PostCreated;
use App/Listeners/NotifyPostCreated;

//...

/**
 * The event listener mappings for the application.
 *
 * @var array
 */
protected $listen = [
    //...
    PostCreated::class => [
        NotifyPostCreated::class,
    ],
];
```

Now that we have setup events and listeners, all that remains is to define some logic inside the event and listener. Then, finally dispatching/raising the event so execute a list of operations contained in each listener.

### Defining Events and Listeners

First, let’s start with the event class **PostCreated** which we created earlier. What we want is to notify users every time a new post is created.

For that, we’ll need to pass the actual `$post` instance to the `__contruct` method of this `PostCreated `class. You’ll learn how to dispatch an event later down but the purpose of this is to make the $post instance accessible among all listeners.

```
# app/Events/PostCreated.php

<?php
 
namespace App\Events;
 
....
use App\Post;
 
class PostCreated
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
 
    public $post;
 
    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct(Post $post)
    {
        $this->post = $post;   
    }
}
```

Second, the listener class `NotifyPostCreated` will hold the actual logic to send the email to the users.

Notice there’s a handle function, this is the method which will get called by the event and within this `handle` method, we’ll write all the necessary code to send the email.

```
# app/Events/NotifyPostCreated.php

<?php
 
namespace App\Listeners;
 
use App\Events\PostCreated;
use App\User;
use Mail;
 
class NotifyPostCreated
{
    ....
    ....
 
    /**
     * Handle the event.
     *
     * @param  PostCreated $event
     * @return void
     */
    public function handle(PostCreated $event)
    {
        // Access the post using $event->post...
        $users = User::all();

        foreach($users as $user) {
           Mail::to($user)->send('emails.post.created', $event->post);
        }
    }
}
```

Please do note that I’ve used `Mail` facade to send an email and configured mail credentials accordingly and also created an email markup blade file in `resources/views/emails/post/created.blade.php`.

## Dispatching Events

The last step is to trigger or fire the `PostCreated` event. All you need to do is use `event()` helper. This helper will dispatch the event to all of its registered `listeners`.

For instance, In PostController:

```
# app/Http/Controllers/PostController.php

<?php

namespace App\Http\Controllers;

use App\Events\PostCreated;
use App\Http\Controllers\Controller;
use App\Post;

class PostController extends Controller
{
    //...

    public function store(Request $request)
    {
        // post storing logic...

        event(new PostCreated($post));
    }
}
```

Alternatively, if your event uses the `Illuminate\Foundation\Events\Dispatchable` trait, you may call the static `dispatch` method on the event like this.

```
//...

public function store(Request $request)
{
    // post storing logic...

    PostCreated::dispatch($post);
}
```

Finally, whenever a new post is created, `PostCreated` event will be raised and eventually executes the listener `NotifyPostCreated` to notify users.

I hope this article made you understand the concept of events and listeners and how you can implement this in your own laravel application.

You can find out more about events and listeners in [laravel docs.](https://laravel.com/docs/7.x/events)

