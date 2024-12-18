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

# Advanced Features

**Custom Time Zones:**

You can specify a custom time zone for tasks that need to run in specific regions.

        $schedule->command('emails:send')->dailyAt('07:00')->timezone('Asia/Dhaka');
This ensures the task runs at 7 AM in the Dhaka timezone.

**Conditional Execution:**

You can run tasks based on certain conditions using the when method.

        $schedule->command('cleanup:files')->daily()->when(function () {
            return now()->isMonday(); // Only run on Mondays
        });


**Notification on Task Completion:**

You can specify actions to take on task success or failure. For example, you can send a notification or log a message when a task is successful.

        $schedule->command('db:backup')->daily()->onSuccess(function () {
            \Log::info('Backup completed successfully');
        });

**Without Overlapping:**

You can ensure that tasks don't overlap by adding withoutOverlapping().

        $schedule->command('emails:send')->everyMinute()->withoutOverlapping();

This prevents the same task from running concurrently if it takes longer than expected to complete.

**Maintenance Mode Exclusion:**

Sometimes, tasks should still run during maintenance mode. You can ensure this with evenInMaintenanceMode().

        $schedule->command('report:generate')->daily()->evenInMaintenanceMode();



# Monitoring Scheduled Tasks

**Task Output Logging:**

You can send the output of the task to a log file for debugging or tracking.

        $schedule->command('emails:send')
                 ->daily()
                 ->sendOutputTo('/path/to/output.log');

The **sendOutputTo()** method in the Laravel Scheduler is used to capture the output (both standard output and errors) of a scheduled command and store it in a specified log file. This is especially useful for debugging or keeping a record of task execution details.

If the command executes successfully:

        Sending emails started...
        Successfully sent 50 emails!
        Task completed at: 2024-12-18 10:00:00

If the command fails:

        Sending emails started...
        Error: Unable to connect to SMTP server!
        Task failed at: 2024-12-18 10:00:00

**Error Tracking**

If you want to log errors separately, you can combine onFailure() with sendOutputTo():

        $schedule->command('emails:send')
                 ->daily()
                 ->sendOutputTo(storage_path('logs/emails.log'))
                 ->onFailure(function () {
                     \Log::error('Email command failed!');
                 });

# Custom Artisan Commands for Scheduling

Artisan commands allow you to define and execute specific tasks through the command line. These commands can be scheduled using the Laravel Scheduler to automate processes like sending emails, generating reports, or cleaning up files.

**Create a Custom Artisan Command**

To create a custom command, use the Artisan make:command generator:

        php artisan make:command SendEmails

This creates a new command file in the app/Console/Commands directory.

**Define the Command Logic**

Open the app/Console/Commands/SendEmails.php file and customize the command.

        <?php
        
        namespace App\Console\Commands;
        
        use Illuminate\Console\Command;
        
        class SendEmails extends Command
        {
            /**
             * The name and signature of the console command.
             *
             * @var string
             */
            protected $signature = 'emails:send';
        
            /**
             * The console command description.
             *
             * @var string
             */
            protected $description = 'Send reminder emails to users';
        
            /**
             * Execute the console command.
             *
             * @return int
             */
            public function handle()
            {
                // Task logic here (e.g., sending emails)
                \Log::info('Reminder emails sent successfully!');
                
                // Example of calling an email service
                // Mail::to(User::all())->send(new ReminderEmail());
        
                return Command::SUCCESS;
            }
        }


**$signature**: This defines how you will run the command. For example, emails:send.
**$description**: This is a short description of what the command does.
**handle()**: This is where the logic for the command resides. In the example, we log a message to indicate the task was executed.

**Register the Command (Optional)**

Commands are automatically registered if they are in the app/Console/Commands directory. However, you can manually register them in the commands array in app/Console/Kernel.php:

        protected $commands = [
            \App\Console\Commands\SendEmails::class,
        ];

**Schedule the Command**

To automate the execution of the command, add it to the schedule() method in app/Console/Kernel.php:

        protected function schedule(Schedule $schedule)
        {
            $schedule->command('emails:send')->weekly();
        }

**emails:send**: Refers to the signature defined in the command.
**->weekly()**: Specifies that the command should run once a week. You can use other scheduling frequencies such as daily(), hourly(), or monthly().

**Test the Command**

To test if your command works as expected, run it manually:

        php artisan emails:send

Check your logs (e.g., storage/logs/laravel.log) for the success message: Reminder emails sent successfully!



