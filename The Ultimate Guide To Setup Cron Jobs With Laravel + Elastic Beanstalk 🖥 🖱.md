# The Ultimate Guide To Setup Cron Jobs With Laravel + Elastic Beanstalk 🖥 🖱



I had quite a big challenge trying to setup Laravel with Elastic Beanstalk EC2 Instances and I tried my best to look for resources explaining the steps in detail but all I found was cluttered information in the internet, so I spent a long time trying and debugging until I got it working at the end 🙌. Therefore, I decided to write this article hoping it may help anyone who might encounter the same struggle I had to save him some time or at least, to be a reference for my self in the future. So, without further ado, let’s get started…

# What is CRON Job 🤔

CRON job is a task that is assigned to some process working in the background to be implemented at a specific time periodically. Having such a requirement is very common in any type of project. For example, you may have an education management system where you need to notify the students about the approach of the deadline. In this case, you need to set a cron job that runs each minute and checks these tasks of which having the deadline < x minutes, and if this condition is true, then you notify the user. Another common example that you may need to do in the server, is to make periodic backups to save a copy of your data. These are just two scenarios but as I mentioned there are too many scenarios where you need to do such a thing in any kind of system.

# How CRON Jobs Work In Linux 💻

Before starting with the actual implementation, I think it’s very fundamental for making the cron jobs or even debugging them to have a basic knowledge of how the cron jobs work in linux.

Basically, there is a service working in the background (daemon) which constantly checks the `/etc/crontab` and `/etc/cron.*` directories and looks for all the **CRON** tasks that are defined in a CRON-task syntax and executes them. In other words, if you need to define cron jobs, then you have to define them in a file inside any of these directories which is what we will be doing in this guide.

# CRON Syntax 🏗

As we said you have to make any file inside the `/etc/cron.*` directory and define the jobs there. Therefore, you can find `/etc/cron.d` directory or create it in your server if not exist and create a file within that directory and with any name (for example: `cron_example`) and finally, define the cron jobs inside that file.

To define any cron job you have to follow this syntax:

```
(min) (hour) (day of month) (month) (day of week) <Username> <your command>
```

As you can see, the first five inputs of any CRON job definition are indicating how often this command should be run. And the format of each of these (timing) inputs should be either a number or one of the acceptable symbols such as:

```
* , - /
```

The most common one of these is the asterisk `*` which is used to define `any` . For example:

```
* * * * * root echo "hello world" > /input.txt
```

this command will be run each minute of each hour of each day of the month of each month and each day of the week which is in short each minute 😅, and the user that will be executing the command is the root user, and what the command does is simply writing the famous wisdom of programming `hello world` in the `/input.txt` file.

Another example could be:

```
15 14 1 * * /path/to/script.sh
```

This will run the `[script.sh](<http://script.sh>)` at 2:15pm on the first of every month.

This is in short how the **CRON** job works and how its syntax is constructed, I think this is enough to grasp the basics and if you are looking for more details, then you can check this nice article:

https://www.cyberciti.biz/faq/how-do-i-add-jobs-to-cron-under-linux-or-unix-oses/

# Building Scheduled Tasks In Laravel 📆 ⏲

Now, it’s the time to get our hands dirty and start working. But before doing so, it’s worth mentioning that: in general, scheduled tasks could be considered same as CRON jobs but more specifically, they are the periodic tasks defined in Laravel-level and CRON jobs are the periodic tasks name in the Linux OS level, and in this guide, we will define a CRON job in linux to call the scheduled tasks in Laravel.

For making scheduled tasks in Laravel you have three main options:

A. Scheduling a queued job.

B. Scheduling an artisan command.

C. Scheduling a function defined directly as parameter.

For A & B you can read more about them in the official Laravel Docs: https://laravel.com/docs/7.x/scheduling

But for the sake of simplicity, I will be using C in this article.

Defining scheduled tasks in Laravel is very handy, and to do so, all of the commands should be defined in `app/Console/Kernel.php` and more specifically, in the `schedule` method.

Let’s have a simple example to start with, let’s say we have a system that has a registration feature where users can register in it (just like 99.99% of the systems in the internet 😁), and we want to show how many new users registered in the system each day by logging this number in the logs file or any defined channel like Slack channel for example.

I won’t be getting into the details of how to link your backend with slack, but here is the link from Laravel docs of logging: https://laravel.com/docs/7.x/logging.

To implement this task, here is how my code looks in the `schedule` function in the `app/Console/Kernel.php` , have a look at it and I will dive into the code details then.

```
protected function schedule(Schedule $schedule)
{
		$schedule->call(function(){
        $today = Carbon::now()->format('yy-m-d');
        $count = Customer::whereDate('created_at', $today)->get()->count();
        Log::channel('data_channel')->info("Total New Customers Today: ".$count);
    })
    ->dailyAt('23:55'); // will be run everyday at 11:55pm 
}
```

As you can see, I just passed the task that I want to implement as a function in the parameter of the `call` function from the `$schedule` object and then defined the time by using the function `dailyAt(<time>)` to specify how often this function will be running.

Note how simple it’s to define the scheduled task in Laravel and how easy it is to specify its repetition as compared to the CRON syntax in Linux.

`dailyAt` is the function that fits with our purpose for this situation, however, it not the only one and you have many other nice functions provided by Laravel to cover most if not all the timing cases as shown in the following image:

![Image for post](https://miro.medium.com/max/60/1*cK4Jn7yGABm7NUTaf255Gw.png?q=20)

![Image for post](https://miro.medium.com/max/1410/1*cK4Jn7yGABm7NUTaf255Gw.png)

for more details: check the docs: https://laravel.com/docs/7.x/scheduling

Now, we have a scheduled task and before pushing our code to the server, we need to make sure that it’s working fine locally, and to do that you can run:

```
php artisan schedule:run
```

if you follow my example above and you use this command then, most probably, you will get an output of: `No scheduled commands are ready to run.` and that's because the task we defined is scheduled at 11:55PM. Well, does that mean you have to wait until 11:55PM and run the `schedule:run` to check if it's working? For sure not. The simplest way to handle this is to change the output temporarily from `dailyAt` to `everyMinute` and return it back to daily once you are ready to push your code to the server. let's make this simple change in our code:

```
protected function schedule(Schedule $schedule)
{
		$schedule->call(function(){
        $today = Carbon::now()->format('yy-m-d');
        $count = Customer::whereDate('created_at', $today)->get()->count();
        Log::channel('data_channel')->info("Total New Customers Today: ".$count);
    })
    ->everyMinute(); // temporarily for testing purposes
}
```

now run `php artisan schedule:run` again and you should get an output like: `Running scheduled command: Closure` which indicates that the task has run successfully. You can check your log files or change the task to do something else like making a new object in the database but you need to check the database then to make sure the command has run without any issues.

We are ready now to push our code to the server and setup the server configuration to run the scheduled tasks. Therefore, you can now return the part of `dailyAt` to your code, however, you can also keep it as `everyMinute` for now to test if it's actually working in the server as well and once your comfortable with it, you can push another version that contains the daily function.

# Setting Up Elastic Beanstalk To Run The Scheduled Tasks 👷

Now, our code is ready to be pushed to the server and the last step to do is to configure the EC2 instances of our elastic beanstalk environment to run our Laravel scheduled tasks. The bad news is that this step took me 3 days to get it working without issues, the good news, however, is that I gathered everything needed to complete the configuration in this article starting from understanding how CRON jobs work which is the first section that I believe is key to setup your Laravel-based cron jobs without issues. To do that, we will make our configuration on the .ebextensions directory which is used for setting up the configuration of the EC2 instances (servers) in the elasticbeanstalk environment. For more details: https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/ebextensions.html

So let’s begin:

According to what we did locally, we ran the command `php artisan schedule:run` but we need to understand how this command work first. according to Laravel docs:

> *When the schedule:run command is executed, Laravel will evaluate your scheduled tasks and runs the tasks that are due.*

Which means running this command will check the current time and compare it with the time of which the scheduled task should be run at, and the tasks that are due will be the ones that will run and whereas the other ones will be ignored until the `schedule:run` command is executed again and the current time (now) exceeds their scheduled time. Hence, we need to add a CRON job in the server that runs in every minute to do only one thing: calling `schedule:run` and the rest will be done by Laravel.

To achieve this, let’s create a file called `cron-setup.config` inside `.ebextension` directory, and in this file we need to add the cron job that calls Laravel's schedule. But what should be the content of this file to implement this configuration? according to Laravel docs:

> *When using the scheduler, you only need to add the following Cron entry to your server.*

```
* * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```

However, this is the tricky part, using this command in your config file won’t run the tasks successfully and there are some missing parts that should be added first. The right entry content that should be added in the case of **elasticbeanstalk** is:

```
* * * * * root . /opt/elasticbeanstalk/support/envvars && /usr/bin/php /var/www/html/artisan schedule:run 1>> /laralog.log 2>&1
```

Let me dive into the details of the command above:

`* * * * *` : is as I explained in the first section: the timing of the called command which in this case every minute.

`root` : I also mentioned this earlier, <username> is part of the entry that should be added which is not mentioned in Laravel docs. In our case the value should be the `root` to have full privileges to run the command.

`. /opt/elasticbeanstalk/support/envvars` : This is the most important part which loads all the environment variables values to be used if needed. Which means if your command is connecting to a slack channel to log some message or even making any read or write from the database and the credentials for doing either of these actions is defined as environment variables and if you don't use this, the task will definitely fail.

`&&`: used for adding another command.

`/usr/bin/php /var/www/html/artisan schedule:run 1>> /laralog.log 2>&1` which is the main command that should run and calls Laravel schedule. It's important to note that I changed the part of `>> /dev/null 2>&1` from Laravel's docs (which is used to write the output of the task in the `dev/null` which is the null file in linux that doesn't show any content) to be: `1>> /laralog.log 2>&1` which is a real file that will contain the output of the task. This step is the one that helped me to debug the task and to find out that the environment variables are empty and should be loaded first.

After all of that, let’s add the required content to our `cron-setup.config` file which should be as follows:

```
"/etc/cron.d/cron_example":
    mode: "000644"
    owner: root
    group: root
    content: |
      * * * * * root . /opt/elasticbeanstalk/support/envvars && /usr/bin/php /var/www/html/artisan schedule:run 1>> /laralog.log 2>&1commands:
  rm_old_cron:
    command: "rm -fr /etc/cron.d/cron_example.bak"
    ignoreErrors: true
```

Note: the `rm_old_cron` inside `commands` is used to remove the old `cron_example` file whenever a new version of the code is deployed.

Now that everything is setup, we are ready to deploy the code into elastic beanstalk instances by calling `eb deploy` and waiting for our tasks to be executed 😄

# One Last Thing 🤞

When you use elastic beanstalk, most probably, you have more than one EC2 instance for sometime (if not all the time), and therefore, you don’t want your scheduled tasks to be executed (n) times where n is the number of EC2 instances. Why would that happen? because the `cron-setup.config` file will be loaded in all EC2 instances and the specified cron job will be added and run in all of them as well, hence, we need to resolve this issue and make sure the scheduled tasks are run only once.

One of the solutions that you can do is to use custom script for only running the CRON jobs from the `leader_only` in **aws** such as the this example shown by **aws** here: https://github.com/awsdocs/elastic-beanstalk-samples/blob/master/configuration-files/aws-provided/instance-configuration/cron-leaderonly-linux.config.

Having that said, one of the great features added to Laravel 5.6 was **Single Server Scheduling** as announced by the creator of Laravel **Taylor Otwell** in this article: https://medium.com/@taylorotwell/laravel-5-6-preview-single-server-scheduling-54df8e0e139b

and to add this to our code, we need to:

1- Use Redis or Memcached as the cache driver based on what’s written in the article.

> *Note: When using this feature, you will need to use the Redis or Memcached cache drivers. These drivers provide the atomicity needed to secure the locks that power this feature.*

2- we need to add the function `onOneServer()` to our scheduled tasks.

That’s it!

I hope this article was helpful and has made it easier to make the complete setup of CRON jobs with Laravel and elastic beanstalk.

Let me know if you have any questions or additions, would appreciate any kind of response. 😃

