# How to add Real-Time Notifications to Laravel with Pusher?

The modern web user expects to be informed of everything that happens within the application. You don’t want to be that one website that doesn’t even have the notifications dropdown found not just in all social media websites, but everywhere else these days, too.

Luckily, with Laravel and [Pusher](https://pusher.com/), implementing this functionality is a breeze.

![Image for post](https://miro.medium.com/max/60/1*WjGGccc3ULu_pkWDd0U8RA.jpeg?q=20)

![Image for post](https://miro.medium.com/max/1280/1*WjGGccc3ULu_pkWDd0U8RA.jpeg)

# Real-Time Notifications

In order to provide users with a good experience, notifications should to be shown in real-time. One approach is to send an AJAX request regularly to the back end and fetch the newest notifications if they exist.

A better approach is to leverage the power of WebSockets, and receive notifications the moment they are sent. This is what we’re going to use in this article.

# Pusher

Pusher is a web service for *integrating realtime bi-directional functionality via WebSockets to web and mobile apps.*

It has a very simple API, but we’re going to make using it even simpler with Laravel Broadcasting and Laravel Echo.

In this article, we’re going to add real-time notifications to an existing blog.

# The Project

# Initialization

First we’ll clone the simple Laravel blog:

```
git clone   https://github.com/marslan-ali/laravel-blog
```

Then we’ll create a MySQL database and set up environment variables to give the application access to the database.

Let’s copy `env.example` to `.env` and update the database related variables.

```
cp .env.example .envDB_HOST=localhost
DB_DATABASE=homestead
DB_USERNAME=homestead
DB_PASSWORD=secret
```

Now let’s install the project’s dependencies with

```
composer install
```

And run the migration and seeding command to populate the database with some data:

```
php artisan migrate --seed
```

If you run the application and visit `/posts` you’ll be able to see a listing of generated posts.
Check the application, register a user, and create some posts. It’s a very basic app, but serves our demo perfectly.

# Follow Users Relationships

We like to give users the ability to follow other users, and be followed by users, so we have to create a `Many To Many` relationship between users to make it happen.

Let’s make a pivot table that relates users to users. Make a new `followers` migration:

```
php artisan make:migration create_followers_table --create=followers
```

We need to add some fields to that migration: a `user_id` to represent the user who is following, and a `follows_id` field to represent the user who’s being followed.

Update the migration as follows:

```
public function up()
{
    Schema::create('followers', function (Blueprint $table) {
        $table->increments('id');
        $table->integer('user_id')->index();
        $table->integer('follows_id')->index();
        $table->timestamps();
    });
}
```

Now let’s migrate to create the table:

```
php artisan migrate
```

Let’s add relationships methods to the `User` model.

```
// ...

class extends Authenticatable
{
    // ...

    public function followers() 
    {
        return $this->belongsToMany(self::class, 'followers', 'follows_id', 'user_id')
                    ->withTimestamps();
    }

    public function follows() 
    {
        return $this->belongsToMany(self::class, 'followers', 'user_id', 'follows_id')
                    ->withTimestamps();
    }
}
```

> ```
> *app/User.php*
> ```

Now that the user model has the necessary relationships, `followers` returns all the followers of a user, and `follows` returns everyone the user is following.

We’ll be needing some helper functions to allow the user to `follow` another user, and to check whether a user `isFollowing` a specific user.

```
// ...

class extends Authenticatable
{
    // ...

    public function follow($userId) 
    {
        $this->follows()->attach($userId);
        return $this;
    }

    public function unfollow($userId)
    {
        $this->follows()->detach($userId);
        return $this;
    }

    public function isFollowing($userId) 
    {
        return (boolean) $this->follows()->where('follows_id', $userId)->first(['id']);
    }

}
```

> ```
> *app/User.php*
> ```

Perfect. With the model set, it’s time to list users.

# Listing Users

Let’s start by setting the necessary routes

```
//...
Route::group(['middleware' => 'auth'], function () {
    Route::get('users', 'UsersController@index')->name('users');
    Route::post('users/{user}/follow', 'UsersController@follow')->name('follow');
    Route::delete('users/{user}/unfollow', 'UsersController@unfollow')->name('unfollow');
});
```

> ```
> *routes/web.php*
> ```

Then, it’s time to create a new controller for users:

```
php artisan make:controller UsersController
```

We’ll add an `index` method to it:

```
// ...
use App\User;
class UsersController extends Controller
{
    //..
    public function index()
    {
        $users = User::where('id', '!=', auth()->user()->id)->get();
        return view('users.index', compact('users'));
    }
}
```

> ```
> *app/Http/Controllers/UsersController.php*
> ```

The method needs a view. Let’s create the `users.index` view and put this markup in it:

```
@extends('layouts.app')

@section('content')
    <div class="container">
        <div class="col-sm-offset-2 col-sm-8">

            <!-- Following -->
            <div class="panel panel-default">
                <div class="panel-heading">
                    All Users
                </div>

                <div class="panel-body">
                    <table class="table table-striped task-table">
                        <thead>
                        <th>User</th>
                        <th> </th>
                        </thead>
                        <tbody>
                        @foreach ($users as $user)
                            <tr>
                                <td clphpass="table-text"><div>{{ $user->name }}</div></td>
                                @if (auth()->user()->isFollowing($user->id))
                                    <td>
                                        <form action="{{route('unfollow', ['id' => $user->id])}}" method="POST">
                                            {{ csrf_field() }}
                                            {{ method_field('DELETE') }}

                                            <button type="submit" id="delete-follow-{{ $user->id }}" class="btn btn-danger">
                                                <i class="fa fa-btn fa-trash"></i>Unfollow
                                            </button>
                                        </form>
                                    </td>
                                @else
                                    <td>
                                        <form action="{{route('follow', ['id' => $user->id])}}" method="POST">
                                            {{ csrf_field() }}

                                            <button type="submit" id="follow-user-{{ $user->id }}" class="btn btn-success">
                                                <i class="fa fa-btn fa-user"></i>Follow
                                            </button>
                                        </form>
                                    </td>
                                @endif
                            </tr>
                        @endforeach
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    </div>
@endsection
```

> ```
> *resources/views/users/index.blade.php*
> ```

You can now visit the `/users` page to see a listing of users.

# To Follow, or to Unfollow

The `UsersController` lacks `follow` and `unfollow` methods. Let’s get them done to wrap this part up.

```
//...
class UsersController extends Controller
{
    //...
    public function follow(User $user)
    {
        $follower = auth()->user();
        if ($follower->id == $user->id) {
            return back()->withError("You can't follow yourself");
        }
        if(!$follower->isFollowing($user->id)) {
            $follower->follow($user->id);

            // sending a notification
            $user->notify(new UserFollowed($follower));

            return back()->withSuccess("You are now friends with {$user->name}");
        }
        return back()->withError("You are already following {$user->name}");
    }

    public function unfollow(User $user)
    {
        $follower = auth()->user();
        if($follower->isFollowing($user->id)) {
            $follower->unfollow($user->id);
            return back()->withSuccess("You are no longer friends with {$user->name}");
        }
        return back()->withError("You are not following {$user->name}");
    }
}
```

> ```
> *app/Http/Controllers/UsersController.php*
> ```

We’re done with the follow functionality. We can now follow and unfollow users from the `/users` page.

# Notifications

Laravel provides an API for sending notifications through multiple channels. Emails, SMS, web notifications, and any other type of notifications can all be sent using the [Notification](https://laravel.com/docs/5.4/notifications) class.

We are going to have two types of notifications:

- Follow notification: sent to a user when they get followed by another user
- Post created notification: sent to the followers of a given user when they create a new post

## User Followed Notification

Using artisan commands, we can generate a migration for notifications:

```
php artisan notifications:table
```

Let’s migrate and create this new table.

```
php artisan migrate
```

We’re starting with follow notifications. Let’s execute this command to generate a notification class:

```
php artisan make:notification UserFollowed
```

Then we’ll update the notification class file we just created:

```
class UserFollowed extends Notification implements ShouldQueue
{
    use Queueable;

    protected $follower;

    public function __construct(User $follower)
    {
        $this->follower = $follower;
    }

    public function via($notifiable)
    {
        return ['database'];
    }

    public function toDatabase($notifiable)
    {
        return [
            'follower_id' => $this->follower->id,
            'follower_name' => $this->follower->name,
        ];
    }
}
```

> ```
> *app/Notifications/UserFollowed.php*
> ```

With these few lines of code we can achieve a lot. First we’re requiring an instance of the `$follower` to be injected when this notification is created.

Using the `via` method, we’re telling Laravel to send this notification via the `database` channel. When Laravel encounters this, it will create a new record in the notifications table.

The `user_id` and notification `type` are automatically set, plus we can extend
the notification with more data. That’s what `toDatabase` is for. The returned array will be added to the `data` field of the notification.

Finally, by implementing `ShouldQueue`, Laravel will automatically put this notification inside a queue to be executed in the background, which will speed up the response. This makes sense because we will be adding HTTP calls when we use Pusher later on.

Let’s initiate the notification when the user gets followed.

```
// ...
use App\Notifications\UserFollowed;
class UsersController extends Controller
{
    // ...
    public function follow(User $user)
    {
        $follower = auth()->user();
        if ( ! $follower->isFollowing($user->id)) {
            $follower->follow($user->id);

            // add this to send a notification
            $user->notify(new UserFollowed($follower));

            return back()->withSuccess("You are now friends with {$user->name}");
        }

        return back()->withSuccess("You are already following {$user->name}");
    }

    //...
}
```

> ```
> *app/Http/Controllers/UsersController.php*
> ```
>
> *Report Advertisement*

We could call the `notify` method on a `User` model because it is already using the [Notifiable](https://laravel.com/docs/5.4/notifications#using-the-notifiable-trait) trait.
Any model you want to notify should be using it to get access to the `notify` method.

## Mark a Notification as Read

Notifications will contain some information and a link to a resource. For example: when a user receives a notification about a new post, the notification should show an informative text, redirect the user to the post when clicked, and be flagged as read.

We’re going to make a middleware that checks if a request has a `?read=notification_id` input and flag it as read.

Let’s make a middleware with the following command:

```
php artisan make:middleware MarkNotificationAsRead
```

Then, let’s put this code inside the `handle` method of the middleware:

```
class MarkNotificationAsRead
{
    public function handle($request, Closure $next)
    {
        if($request->has('read')) {
            $notification = $request->user()->notifications()->where('id', $request->read)->first();
            if($notification) {
                $notification->markAsRead();
            }
        }
        return $next($request);
    }
}
```

> ```
> *app/Http/Middleware/MarkNotificationAsRead.php*
> ```

In order to get our middleware to be executed for each request, we’ll add it to `$middlewareGroups`.

```
//...
class Kernel extends HttpKernel
{
    //...
    protected $middlewareGroups = [
        'web' => [
            //...            \App\Http\Middleware\MarkNotificationAsRead::class,
        ],
        // ...
    ];
    //...
}
```

> ```
> *app/Http/Kernel.php*
> ```

With that done, let’s show some notifications.

## Showing Notifications

We have to show a listing of the notifications using AJAX, then update it in real time with Pusher. First, let’s add a `notifications` method to the controller:

```
// ...
class UsersController extends Controller
{
    // ...
    public function notifications()
    {
        return auth()->user()->unreadNotifications()->limit(5)->get()->toArray();
    }
}
```

> ```
> *app/Http/Controllers/UsersController.php*
> ```

This will return the last 5 unread notifications. We just have to add a route to make it accessible.

```
//...
Route::group([ 'middleware' => 'auth' ], function () {
    // ...
    Route::get('/notifications', 'UsersController@notifications');
});
```

> ```
> *routes/web.php*
> ```
>
> *Report Advertisement*

Now add a dropdown for notifications in the header.

```
<head>
    <!-- // ... // -->    <!-- Scripts -->
    <script>
        window.Laravel = <?php echo json_encode([
            'csrfToken' => csrf_token(),
        ]); ?>
    </script>    <!-- This makes the current user's id available in javascript -->
    @if(!auth()->guest())
        <script>
            window.Laravel.userId = <?php echo auth()->user()->id; ?>
        </script>
    @endif
</head>
<body>
    <!-- // ... // -->
    @if (Auth::guest())
        <li><a href="{{ url('/login') }}">Login</a></li>
        <li><a href="{{ url('/register') }}">Register</a></li>
    @else
        <!-- // add this dropdown // -->
        <li class="dropdown">
            <a class="dropdown-toggle" id="notifications" data-toggle="dropdown" aria-haspopup="true" aria-expanded="true">
                <span class="glyphicon glyphicon-user"></span>
            </a>
            <ul class="dropdown-menu" aria-labelledby="notificationsMenu" id="notificationsMenu">
                <li class="dropdown-header">No notifications</li>
            </ul>
        </li>
<!-- // ... // -->
```

> ```
> *resources/views/layouts/app.blade.php*
> ```

We’ve also added a global `window.Laravel.userId` variable inside a script to get the current user’s ID.

## JavaScript and SASS

We’re going to use Laravel Mix to compile JavaScript and SASS. First, we need to install npm packages.

```
npm install
```

Now let’s add this code into `app.js`:

```
window._ = require('lodash');
window.$ = window.jQuery = require('jquery');
require('bootstrap-sass');var notifications = [];const NOTIFICATION_TYPES = {
    follow: 'App\\Notifications\\UserFollowed'
};
```

> ```
> *app/resources/assets/js/app.js*
> ```

This is just an initialization. We’re going to use `notifications` to store all notification objects whether they’re retrieved via AJAX or Pusher.

You probably guessed it, `NOTIFICATION_TYPES` contains types of notifications.

Next, let’s “GET” notifications via AJAX.

```
//...
$(document).ready(function() {
    // check if there's a logged in user
    if(Laravel.userId) {
        $.get('/notifications', function (data) {
            addNotifications(data, "#notifications");
        });
    }
});function addNotifications(newNotifications, target) {
    notifications = _.concat(notifications, newNotifications);
    // show only last 5 notifications
    notifications.slice(0, 5);
    showNotifications(notifications, target);
}
```

> ```
> *app/resources/assets/js/app.js*
> ```

With this, we’re getting the latest notifications from our API and putting them inside the dropdown.

Inside `addNotifications` we concatenate the present notifications with the new ones using [Lodash](https://www.sitepoint.com/tag/lodash/), and take only the latest 5 to be shown.

We need a few more functions to finish the job.

Report Advertisement

```
//...
function showNotifications(notifications, target) {
    if(notifications.length) {
        var htmlElements = notifications.map(function (notification) {
            return makeNotification(notification);
        });
        $(target + 'Menu').html(htmlElements.join(''));
        $(target).addClass('has-notifications')
    } else {
        $(target + 'Menu').html('<li class="dropdown-header">No notifications</li>');
        $(target).removeClass('has-notifications');
    }
}
```

> ```
> *app/resources/assets/js/app.js*
> ```

This function builds a string of all notifications and puts it inside the dropdown.
If no notifications were received, it just shows “No notifications”.

It also adds a class to the dropdown button, which will just change its color when notifications exist. It’s a bit like Github’s notifications.

Finally, some helper functions to make notification strings.

```
//...// Make a single notification string
function makeNotification(notification) {
    var to = routeNotification(notification);
    var notificationText = makeNotificationText(notification);
    return '<li><a href="' + to + '">' + notificationText + '</a></li>';
}// get the notification route based on it's type
function routeNotification(notification) {
    var to = '?read=' + notification.id;
    if(notification.type === NOTIFICATION_TYPES.follow) {
        to = 'users' + to;
    }
    return '/' + to;
}// get the notification text based on it's type
function makeNotificationText(notification) {
    var text = '';
    if(notification.type === NOTIFICATION_TYPES.follow) {
        const name = notification.data.follower_name;
        text += '<strong>' + name + '</strong> followed you';
    }
    return text;
}
```

> ```
> *app/resources/assets/js/app.js*
> ```

Now we’ll just add this to our `app.scss` file:

```
//... 
#notifications.has-notifications {
  color: #bf5329
}
```

> ```
> *app/resources/assets/sass/app.scss*
> ```

Let’s compile assets:

```
npm run dev
```

If you try and follow a user now, they’ll get a notification. When they click it, they’ll be redirected to `/users`, plus the notification will disappear.

# New Post Notification

We’re going to notify followers when a user creates a new post.

Let’s start by generating the notification class.

```
php artisan make:notification NewPost
```

Let’s update the generated class as follows:

```
// ..
use App\Post;
use App\User;
class NewArticle extends Notification implements ShouldQueue
{
    // ..
    protected $following;
    protected $post;    public function __construct(User $following, Post $post)
    {
        $this->following = $following;
        $this->post = $post;
    }    public function via($notifiable)
    {
        return ['database'];
    }    public function toDatabase($notifiable)
    {
        return [
            'following_id' => $this->following->id,
            'following_name' => $this->following->name,
            'post_id' => $this->post->id,
        ];
    }
}
```

> ```
> *app/Notifications/NewArticle.php*
> ```
>
> *Report Advertisement*

Next, we need to send the notification. There are several ways we could do this.
I like to use Eloquent Observers.

Let’s make an observer for `Post` and listen to its events. We’ll create a new class: `app/Observers/PostObserver.php`

```
namespace App\Observers;use App\Notifications\NewPost;
use App\Post;class PostObserver
{
    public function created(Post $post)
    {
        $user = $post->user;
        foreach ($user->followers as $follower) {
            $follower->notify(new NewPost($user, $post));
        }
    }
}
```

Then, register the observer in `AppServiceProvider`:

```
//...
use App\Observers\PostObserver;
use App\Post;class AppServiceProvider extends ServiceProvider
{
    //...
    public function boot()
    {
        Post::observe(PostObserver::class);
    }
    //...
}
```

> ```
> *app/Providers/AppServiceProvider.php*
> ```

Now we just need to format the message to be shown in JS:

```
// ...
const NOTIFICATION_TYPES = {
    follow: 'App\\Notifications\\UserFollowed',
    newPost: 'App\\Notifications\\NewPost'
};
//...
function routeNotification(notification) {
    var to = `?read=${notification.id}`;
    if(notification.type === NOTIFICATION_TYPES.follow) {
        to = 'users' + to;
    } else if(notification.type === NOTIFICATION_TYPES.newPost) {
        const postId = notification.data.post_id;
        to = `posts/${postId}` + to;
    }
    return '/' + to;
}function makeNotificationText(notification) {
    var text = '';
    if(notification.type === NOTIFICATION_TYPES.follow) {
        const name = notification.data.follower_name;
        text += `<strong>${name}</strong> followed you`;
    } else if(notification.type === NOTIFICATION_TYPES.newPost) {
        const name = notification.data.following_name;
        text += `<strong>${name}</strong> published a post`;
    }
    return text;
}
```

> ```
> *app/resources/assets/js/app.js*
> ```

And voilà! Users are getting notifications about follows and new posts! Go ahead and try it out!

# Going Real-Time with Pusher

It’s time to use Pusher to get notifications in real-time through websockets.

Sign up for a free Pusher account at [pusher.com](https://pusher.com/) and create a new app.

```
...BROADCAST_DRIVER=pusher
PUSHER_KEY=
PUSHER_SECRET=
PUSHER_APP_ID=
```

Set your account’s options inside the `broadcasting` config file:

```
//...
    'connections' => [            'pusher' => [
                //...
                'options' => [
                    'cluster' => 'eu',
                    'encrypted' => true
                ],
            ],
    //...
```

> ```
> *config/broadcasting.php*
> ```

Then we’ll register `App\Providers\BroadcastServiceProvider` in the providers array.

```
// ...
'providers' => [
    // ...
    App\Providers\BroadcastServiceProvider
    //...
],
//...
```

> ```
> *config/app.php*
> ```
>
> *Report Advertisement*

We should install Pusher’s PHP SDK and Laravel Echo now:

```
composer require pusher/pusher-php-servernpm install --save laravel-echo pusher-js
```

We have to set the notification data to be broadcast. Let’s update the `UserFollowed` notification:

```
//...
class UserFollowed extends Notification implements ShouldQueue
{
    // ..
    public function via($notifiable)
    {
        return ['database', 'broadcast'];
    }
    //...
    public function toArray($notifiable)
    {
        return [
            'id' => $this->id,
            'read_at' => null,
            'data' => [
                'follower_id' => $this->follower->id,
                'follower_name' => $this->follower->name,
            ],
        ];
    }
}
```

> ```
> *app/Notifications/UserFollowed.php*
> ```

And `NewPost`:

```
//...
class NewPost extends Notification implements ShouldQueue
{    //...
    public function via($notifiable)
    {
        return ['database', 'broadcast'];
    }
    //...
    public function toArray($notifiable)
    {
        return [
            'id' => $this->id,
            'read_at' => null,
            'data' => [
                'following_id' => $this->following->id,
                'following_name' => $this->following->name,
                'post_id' => $this->post->id,
            ],
        ];
    }
}
```

> ```
> *app/Notifications/NewPost.php*
> ```

The last thing we need to do is update our JS. Open `app.js` and add the following code

```
// ...
window.Pusher = require('pusher-js');
import Echo from "laravel-echo";window.Echo = new Echo({
    broadcaster: 'pusher',
    key: 'your-pusher-key',
    cluster: 'eu',
    encrypted: true
});var notifications = [];//...$(document).ready(function() {
    if(Laravel.userId) {
        //...
        window.Echo.private(`App.User.${Laravel.userId}`)
            .notification((notification) => {
                addNotifications([notification], '#notifications');
            });
    }
});
```

> ```
> *app/resources/assets/js/app.js*
> ```

And we’re done here. Notifications are being added in real-time. You can now play with the app and see how notifications get updated.

# Conclusion

Pusher has a very simple API that makes receiving real-time events incredibly easy. Coupled with Laravel notifications, we could send a notification through multiple channels (email, SMS, Slack, etc.) from one place. In this tutorial, we added user-following functionality to a simple blog, and enhanced it with the aforementioned tools to get some smooth real-time functionality.

There’s a lot more to Pusher and to Laravel notifications: in tandem, the services allow you to send pub/sub messages in real time to browsers, mobiles, and IOT devices. There’s also a presence API to get online/offline status of users.

Please check their respective documentations ([Pusher docs](https://pusher.com/docs), [Pusher tutorials](https://pusher.com/tutorials), [Laravel docs](https://laravel.com/docs/5.4/notifications)) to explore them in more depth and utilize their true potential.

*If you need help in any kind of development project & I can also provide you consultancy about your project. I am top rated freelancer. You can hire me directly on* [***Upwork\***](https://www.upwork.com/fl/muhammadarslanali2)*.* *You can also hire me on* [***Freelancer\***](https://www.freelancer.com/u/ShayanSolutions?w=f)*.*

*If you have any comment, question, or recommendation, feel free to post them in the comment section below!*