# Cron Jobs

Cron jobs are time-based tasks that run automatically on a server. In Linux-based systems, the cron daemon schedules these tasks using the crontab configuration.

Example: Run a script every hour:

    0 * * * * php /path-to-your-project/artisan schedule:run >> /dev/null 2>&1

# Scheduler 

**What is Scheduler?**

The Scheduler is a powerful, built-in tool that allows you to schedule recurring tasks in your Laravel application. It acts as a wrapper around cron jobs, which are traditionally used to run scheduled tasks on a server.

The benefit of using the Laravel Scheduler is that it allows you to define your tasks in a more expressive and cleaner way using PHP, rather than directly managing them through a cron configuration.

**How does Laravel Scheduler work?**

Laravel Scheduler uses a single cron job to invoke the Laravel command scheduler (php artisan schedule:run) every minute. You define your scheduled tasks in the app/Console/Kernel.php file, which is where all the scheduled tasks are registered.

# Setting Up the  Scheduler

**Step 1: Add a Cron Job**

To make the scheduler run every minute, you need to add a cron job to your server. This cron job will call Laravel's schedule:run command, which checks the Kernel.php file for any tasks that need to run.

SSH into your server and open your crontab file:

    crontab -e

Add the following cron entry to run the scheduler every minute:

    * * * * * php /path-to-your-project/artisan schedule:run >> /dev/null 2>&1
    
+ " * * * * * " tells cron to run the task every minute.
+ php /path-to-your-project/artisan schedule:run executes the Laravel scheduler.
+ " >> /dev/null 2>&1 discards the output and error logs.
  
This ensures that the Laravel scheduler is running in the background every minute.

**Step 2: Define Your Schedule**

Once you've set up the cron job, you can define the tasks to be run by the scheduler.

+ Open the app/Console/Kernel.php file.
+ Inside the schedule method, you can register your tasks:

    protected function schedule(Schedule $schedule)
    {
        // Example: Run a command every day at 1 PM
        $schedule->command('your:command')->dailyAt('13:00');
    }

Here, the command('your:command') tells the scheduler to execute a custom Artisan command you have created. The dailyAt('13:00') tells it to run every day at 1 PM.

# Scheduling Common Tasks

**Run Artisan Commands:**

Laravel allows you to schedule built-in Artisan commands or custom ones that you've defined.

For example, to run the queue worker every five minutes:

    $schedule->command('queue:work --stop-when-empty')->everyFiveMinutes();

**Call a Closure:**

You can define a task using a closure. This is useful for running small custom code within your application.

    $schedule->call(function () {
        \Log::info('Task executed at ' . now());
    })->hourly();

This will log a message every hour.

**Execute Shell Commands:**

You can schedule shell commands to run directly from Laravel.

    $schedule->exec('cp storage/logs/*.log storage/backup/')->daily();

This command copies log files to a backup folder every day.

**Ping a URL:**

You can schedule HTTP requests using the ping() method.

    $schedule->ping('https://example.com/health-check')->everyThirtyMinutes();

This would ping the given URL every 30 minutes, useful for uptime monitoring.






