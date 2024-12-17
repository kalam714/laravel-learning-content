# Queues and Job Processing

queue system provides a robust mechanism for deferring time-consuming tasks such as sending emails, processing uploads, and performing heavy computations, allowing your application to respond more quickly to users

**What is a Queue?**

A queue is a mechanism to handle tasks asynchronously. Instead of executing tasks immediately (which might slow down the application), tasks are pushed into a queue and processed in the background.

**Basic Steps to Set Up Queues in Laravel:**

**Install Queue Driver**: Laravel supports multiple queue backends, including database, Redis, Beanstalkd, and Amazon SQS. The default driver is sync, which executes the job immediately. To use a different driver, configure it in .env.

Example for Redis:

    QUEUE_CONNECTION=redis

**Queue Configuration**: The queue configuration file is located at config/queue.php. You can change the connection type and other settings there, depending on the queue service you're using.
**Creating Jobs**: Laravel makes it easy to create jobs using Artisan commands.

    php artisan make:job SendEmail
This generates a SendEmail job in the App/Jobs directory. Jobs contain the logic you want to run in the background.
**Job Structure**: A typical job class looks like this:

    namespace App\Jobs;
    
    use Mail;
    use App\Mail\WelcomeMail;
    
    class SendEmail extends Job
    {
        public $user;
    
        public function __construct($user)
        {
            $this->user = $user;
        }
    
        public function handle()
        {
            Mail::to($this->user->email)->send(new WelcomeMail($this->user));
        }
    }

**Dispatching Jobs**: Once a job is created, you can dispatch it to the queue.

    SendEmail::dispatch($user);
**Queue Worker**: To start processing queued jobs, you need to run a queue worker. You can do this via Artisan:

    php artisan queue:work
This will continuously listen for jobs on the queue and process them as they come in.

#  Queue Management

**Delayed Jobs**: Sometimes, you may want to delay the processing of a job for a certain period. Laravel allows you to specify a delay when dispatching a job

    SendEmail::dispatch($user)->delay(now()->addMinutes(10));
    // delays the job execution by 10 minutes.

**Retrying Jobs**: If a job fails, you can configure Laravel to automatically retry the job a certain number of times. This is especially useful for jobs that might fail temporarily, like network requests.

**Job Retry Logic**:You can specify how many times a job should be retried using the tries property in the job class.
    
    class SendEmail extends Job
    {
        public $tries = 5;
    
        public function handle()
        {
            // Job logic
        }
    }

This will cause the job to be retried up to 5 times before it is marked as failed.

**Delay Between Retries**:You can define a delay between retries by setting the **retryAfter** property.

    class SendEmail extends Job
    {
        public $retryAfter = 30; // Retry after 30 seconds
    }

**Failed Jobs**: 

Laravel provides built-in support for failed jobs. When a job fails after the maximum number of retries, it is moved to a failed_jobs

**Creating the Failed Jobs Table**: Run the following Artisan command to create the migration for failed jobs:
    php artisan queue:failed-table
    php artisan migrate

**Handling Failed Jobs**: You can use the **failed()** method in your job class to define custom behavior when a job fails:

    class SendEmail extends Job
    {
        public function failed(Exception $exception)
        {
            // Log the failure or send an alert
        }
    }

**Viewing Failed Jobs**: You can view the list of failed jobs using Artisan:

    php artisan queue:failed

**Retrying Failed Jobs**: If you want to retry a failed job, you can use Artisan:

    php artisan queue:retry {job_id}
    
    //Or you can retry all failed jobs
    
    php artisan queue:retry all


# Job Batching

**Batch Jobs**:

Laravel allows you to group multiple jobs into a batch and process them together. This feature is useful when you have a collection of jobs that need to be run sequentially, but you still want to benefit from parallel processing.

    use Illuminate\Bus\Batch;
    use Illuminate\Support\Facades\Bus;
    
    $batch = Bus::batch([
        new SendEmail($user),
        new ProcessImage($image),
    ])->dispatch();
You can also monitor the progress of a batch and receive notifications once all jobs are processed.

**Handling Batch Failures**:You can define actions to take if any of the jobs in a batch fail

    $batch->catch(function (Batch $batch, \Throwable $e) {
        // Handle failure
    });

# Queue Priorities and Job Middleware

**Queue Priorities**:

If you have multiple queues or jobs, you can prioritize them. For example, you can process jobs in the high queue before jobs in the low queue.

**Configure Multiple Queues**:Define different queues for different types of jobs in your .env file.

    QUEUE_CONNECTION=redis

**Specifying Queue for a Job**: When dispatching a job, specify which queue it should be placed in:

    SendEmail::dispatch($user)->onQueue('high');
    
**Worker Listening to Specific Queues**: You can specify which queues a worker should listen to:

    php artisan queue:work redis --queue=high,low

**Job Middleware:**

Job middleware allows you to wrap job execution with custom logic (e.g., logging, throttling).

**Creating Middleware**: Middleware can be used to control how jobs are processed before or after the job itself executes.

    class CheckIfUserIsActive
    {
        public function handle($job, $next)
        {
            if ($job->user->is_active) {
                $next($job);
            } else {
                $job->delete();
            }
        }
    }

**Attach Middleware to Jobs**: You can attach middleware to your jobs:

    class SendEmail extends Job
    {
        public function middleware()
        {
            return [new CheckIfUserIsActive];
        }
    
        public function handle()
        {
            // Job logic
        }
    }

# Queue Monitoring and Optimization

Use tools like Laravel Horizon to monitor your queues in real-time. Horizon provides a beautiful dashboard for managing and monitoring jobs, including metrics like throughput, failures, and processing time.

**Install Laravel Horizon**: Horizon is installed via Composer:

    composer require laravel/horizon
    php artisan horizon:install
    php artisan migrate

**Running Horizon**: Start Horizon with:

    php artisan horizon

You can access the Horizon dashboard by navigating to /horizon in your browser.

**Optimizing Queues**

**Queue Workers in Daemons**: For production systems, queue workers should run as daemons, meaning they should run continuously and restart automatically. Use tools like Supervisor to manage queue workers.

Example Supervisor configuration:

    [program:queue-worker]
    process_name=%(program_name)s
    command=php /path/to/artisan queue:work redis --daemon
    autostart=true
    autorestart=true
    user=www-data
    redirect_stderr=true
    stdout_logfile=/path/to/storage/logs/worker.log

**Optimizing Job Performance**:

+ Ensure that your jobs are small and focused. Avoid performing too many actions within a single job.
+ Use chunking to process large datasets efficiently without overwhelming the worker’s memory.

** Real-life scenarios where queues and workers are commonly used:**

**Sending Emails**

**Scenario**:
When a user registers or makes a purchase, your application sends them a confirmation email.

**Why use a queue?**
Sending an email involves external SMTP servers, which may delay the response. Using a queue, the email can be sent in the background while the user gets immediate feedback.

        Mail::to($user->email)->queue(new WelcomeEmail($user));

**Processing File Uploads**

**Scenario**:

A user uploads a large file (e.g., a video or document) that needs to be processed (e.g., resizing images, converting formats, or scanning for malware).

**Why use a queue?**
Processing files can be resource-intensive and slow. Using a queue ensures the user doesn't wait for the task to finish.

**Example:**

+ Video transcoding for streaming platforms.
+ Generating thumbnails for uploaded images.




**Summary:**

+ Basic Queue Usage: Simple job dispatch and queue workers.
+ Advanced Queue Features: Delayed jobs, retry logic, job failure handling, and batch processing.
+ Job Middleware: Custom logic to wrap job execution.
+ Queue Prioritization: Managing multiple queues with different priorities.
+ Horizon and Monitoring: Real-time monitoring and optimization with Laravel Horizon.
