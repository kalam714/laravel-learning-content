# Bad Practice

Bad practices are ways of writing code that can cause problems in the future. These problems might include security risks, slow performance, or bugs that are hard to fix. Sometimes, the issues only show up years later, especially when new developers join or the project grows. By then, fixing them can take a lot of time and effort.

In this article, I’ll break these down into two groups:

  + Really Bad Practices – These are the ones that can cause big problems like security issues or slow performance.
  + Not-So-Bad Practices – These are less serious but can still make development harder or slower. Some people might even disagree about these—so share your thoughts in the comments!
    
Let’s get started and look at what to avoid in your code!

# Not Preventing N+1 Query Problems

Let’s start with a common issue—N+1 queries.
This happens when your code makes one query to get a list of data (like all books) and then runs additional queries for each item in that list (like getting the author for each book).
Here’s an example of the problem:

    use App\Models\Order;
    
    $orders = Order::all();
    
    foreach ($orders as $order) {
        echo $order->customer->name;
    }
This code will first get all orders in one query, but for every order, it will run a new query to get the customer. If you have 100 orders, that’s 101 queries!

**How to Handle This Problem**
You can fix N+1 queries in three ways:

**Find & Fix Them**: Use tools like Laravel Debugbar to see how many queries are running.

**Avoid Them**: Use Eager Loading to load related data in advance. For example:


    $orders = Order::with('customer')->get(); // Loads customers in the same query
    foreach ($orders as $order) {
        echo $order->customer->name;
    }

**Prevent Them**: Use Laravel's preventLazyLoading() method in development to catch these issues early:

    Model::preventLazyLoading();
By addressing N+1 queries, you can make your app faster and avoid unnecessary database load.


# Loading Too Much Data from the Database

This is another issue that can slow down your app. It happens when you load more data than you actually need, which wastes resources.

Here’s an example of the problem:

    use App\Models\Product;
    
    $products = Product::with('reviews')->get();
    
    foreach ($products as $product) {
        echo $product->name . ': ' . $product->reviews()->count();
    }

**What’s wrong here?**
We’re loading all the reviews for each product, but we only need the total count of reviews.

**The Better Way**
Instead, use withCount() to get the number of reviews without loading all the review data:

    $products = Product::withCount('reviews')->get();
    
    foreach ($products as $product) {
        echo $product->name . ': ' . $product->reviews_count;
    }

**Another Problem**: 
In the same example, we’re loading all columns from the products table, but we only need the name. If the table has large data fields, like descriptions or images, it can slow things down.

Fix this by selecting only the columns you need:

    $products = Product::select('name')
        ->withCount('reviews')
        ->get();
    
    foreach ($products as $product) {
        echo $product->name . ': ' . $product->reviews_count;
    }

**Why It’s Important**
  + Saves Database Resources: Loading unnecessary data puts extra load on your database.
  + Reduces Memory Usage: Large datasets can use up your server’s memory, causing it to slow down or crash.
  + Improves Performance: Smaller, targeted queries make your app faster.
    
**Simple Rule**: Only load the data you actually need. This keeps your app efficient and avoids future problems.

# Chaining Eloquent Without Checking

Have you ever seen something like this in your Blade template?
    {{ $order->customer->email }}
Here, the Order model is connected to the Customer model, so it looks fine. But what happens if the customer is missing, maybe due to deletion or some mistake? You’ll get an error!

**Here’s how to fix it:**

The simplest solution is to use PHP’s null-safe operator:
    {{ $order->customer?->email }}
This checks if the customer exists before trying to get the email. If it doesn’t exist, it simply returns null instead of throwing an error.

**Another Example**
Let’s say you’re trying to get an admin email:

    $adminEmail = User::where('is_admin', true)->first()->email;
What happens if no admin user exists? The first() method will return null, and trying to access email will cause an error.

**The better way:**
Check if the record exists first:
    $admin = User::where('is_admin', true)->first();
    
    if ($admin) {
        $adminEmail = $admin->email;
    } else {
        $adminEmail = 'No admin found';
    }
**Why It’s Important**
  + Prevents Errors: Avoids runtime errors when data is missing.
  + More Reliable: Makes your code future-proof and handles unexpected situations gracefully.
    
**Simple Rule**: Always check if the data exists before accessing it. It’s a small step that saves big headaches later.

# Hardcoding Values Instead of Configs
When you hardcode values (like numbers, strings, or other constants) directly in your code, you make your application less flexible and harder to maintain. If you need to change that value later, you might have to search through multiple files, risking missing some occurrences. Additionally, hardcoded values can cause issues in environments where settings need to differ (e.g., development, staging, production).

Imagine you need to apply a tax rate in a shopping cart calculation:
    $taxRate = 0.15;  // Hardcoded tax rate
    $totalWithTax = $subtotal + ($subtotal * $taxRate);

This works fine at first, but what happens when:
  + Tax Rate Changes: The government increases the tax rate to 18%. You now have to locate all occurrences of 0.15 in your code and update it.
  + Environment-Specific Rates: Your production and development environments require different tax rates (e.g., 15% in production and 10% in development).
  + Error Risk: You could accidentally update the wrong 0.15, leading to bugs.

**Fix: Using Config Files**
Open the config/app.php file (or create a new config file, e.g., config/tax.php) and add the tax rate there:
    'tax_rate' => 0.15,
Instead of hardcoding, use the config() helper:
    $taxRate = config('app.tax_rate');  // Access the tax rate from config
    $totalWithTax = $subtotal + ($subtotal * $taxRate);

# Overloading Controllers

When controllers in Laravel take on too many responsibilities, it becomes harder to maintain, test, and scale your application. A controller's primary job is to handle HTTP requests, but if it starts managing complex business logic, interacting with external services, or handling intricate database operations, it can become bloated.

This leads to several issues:

  + Difficult to Maintain: A controller with too much logic becomes hard to maintain and understand. If you need to modify one part of the logic, you risk breaking other parts.
  + Poor Testability: With many responsibilities inside a controller, writing unit tests becomes complicated because you're testing a large chunk of code rather than isolated functionality.
  + Reduced Reusability: Logic that is tied up in a controller can’t be easily reused elsewhere, making it harder to implement changes across the app without duplicating code.

Imagine a controller that handles user registration, sends a welcome email, processes payment, and logs activities all in one place:

    class UserController extends Controller
    {
        public function register(Request $request)
        {
            // Validate user input
            $validated = $request->validate([
                'name' => 'required|string',
                'email' => 'required|email',
                'password' => 'required|min:8',
            ]);
    
            // Create the user
            $user = User::create([
                'name' => $validated['name'],
                'email' => $validated['email'],
                'password' => bcrypt($validated['password']),
            ]);
    
            // Send a welcome email
            Mail::to($user)->send(new WelcomeEmail($user));
    
            // Process payment
            PaymentService::process($user);
    
            // Log the activity
            Log::info('New user registered', ['user_id' => $user->id]);
    
            return response()->json(['message' => 'User registered successfully']);
        }
    }

This register() method is responsible for:
  + Validating input.
  + Creating a user.
  + Sending an email.
  + Processing payment.
  + Logging the activity.
This is too much logic for a controller to handle. Each of these steps could be isolated into different classes or services.

**Fix:**
The solution is to move logic out of the controller and into separate classes, such as Service Classes, Helpers, or Jobs, depending on the task. This helps keep the controller focused only on handling HTTP requests and responses.

**Ignoring Query Optimization**
When working with large datasets in Laravel, using ->get() to fetch all records from a database at once can lead to performance issues. This is especially problematic if you have tables with thousands or millions of rows. The ->get() method retrieves all data and loads it into memory, which can cause high memory usage, slow performance, and even crash the server if the dataset is large enough.

For example, imagine you're fetching all records from a table without considering the size of the dataset:
    $users = User::all(); // or User::get();
This would load all the users into memory at once, and if there are a large number of users, this can cause the application to run out of memory or perform slowly.

**Fix: Use ->chunk() for Large Datasets:**
To efficiently handle large datasets, you can use the ->chunk() method. This method allows you to retrieve and process smaller portions (chunks) of the data at a time, instead of fetching everything all at once. This reduces memory usage and improves performance.

The ->chunk() method works by splitting the dataset into smaller batches, processing each batch one at a time, and then releasing it from memory once it's done. This makes it suitable for tasks like sending emails to all users or processing large records in the database.

    // Process 100 users at a time
    User::chunk(100, function ($users) {
        foreach ($users as $user) {
            // Process each user, like sending emails
            Mail::to($user)->send(new WelcomeEmail($user));
        }
    });

Use ->chunkById() if you're processing data in batches but need to ensure that records are processed in a sequential manner (i.e., without duplicates or gaps).

    // Using chunkById() ensures no duplicates or missed records
    User::chunkById(100, function ($users) {
        foreach ($users as $user) {
            // Process user
        }
    });

To avoid overloading your server's memory and causing performance issues, don't use ->get() for large datasets. Instead, use ->chunk(), which processes smaller portions of data at a time. This leads to better memory management, faster performance, and makes your application more scalable.

**chunk()) is more suited for background processing tasks where you want to process data in smaller pieces without returning them to the user, and it handles the fetching of the next batch automatically.**

# Using Plain Arrays for Data Structures
developers sometimes use plain arrays to store and manipulate data. While plain arrays can work for basic data storage, they lack many useful features that Laravel's Collections provide. Collections are powerful objects that extend PHP arrays and offer a range of helpful methods to perform operations on data more easily and efficiently.

For example, if you were using a plain array to filter and sort data, you'd have to write custom logic or use basic PHP functions like array_filter and array_map:
    // Plain Array Example
    $users = [
        ['name' => 'Alice', 'age' => 25],
        ['name' => 'Bob', 'age' => 30],
        ['name' => 'Charlie', 'age' => 35],
    ];
    
    // Filtering and sorting manually
    $filtered = array_filter($users, function ($user) {
        return $user['age'] > 28;
    });
    
    usort($filtered, function ($a, $b) {
        return $a['age'] <=> $b['age'];
    });
    
    print_r($filtered);

This approach works, but it's less readable, harder to maintain, and doesn't take full advantage of Laravel's built-in functionalities.

**Fix: Use Laravel Collections for Cleaner, Chainable Operations**

Laravel provides Collections, which are objects that wrap around arrays and come with a rich set of methods for manipulating data. These methods are chainable, meaning you can apply multiple operations on the collection in a more elegant, readable, and efficient way.

Using Collections, the same logic can be expressed much more cleanly:

    use Illuminate\Support\Collection;
    
    $users = collect([
        ['name' => 'Alice', 'age' => 25],
        ['name' => 'Bob', 'age' => 30],
        ['name' => 'Charlie', 'age' => 35],
    ]);
    
    // Using Collection methods to filter and sort
    $filtered = $users->filter(function ($user) {
        return $user['age'] > 28;
    })->sortBy('age');
    
    dd($filtered->values()->all());

In this example:
  + collect(): Wraps the plain array in a Collection.
  + filter(): Filters users based on age, just like array_filter, but using a more expressive and readable method.
  + sortBy(): Sorts the filtered users by age, similar to usort(), but with cleaner syntax.
  + values(): Reindexes the collection after sorting (removes any gaps in array keys).
  + all(): Converts the collection back to an array.

**Why Use Collections Over Plain Arrays?**

**Chainable Methods:** Collections offer a fluent, chainable interface. This means you can perform multiple operations on a collection in a single, readable line of code, rather than calling multiple functions.

    $users = collect($users)
        ->filter(fn($user) => $user['age'] > 28)
        ->sortBy('age')
        ->pluck('name');

**More Built-in Methods:** Collections come with many built-in methods that simplify common tasks, such as:

  + map(), filter(), reduce(), sum(), avg(), groupBy(), pluck(), and many more.
  + These methods allow for more complex operations with less code.

**Better Readability:** Code using Collections tends to be more readable and easier to maintain. Methods like filter(), map(), and sortBy() clearly convey what is happening, unlike the more generic array_filter() or usort() functions.
**Improved Performance:** Laravel Collections are optimized for performance and memory usage. While plain arrays are basic data structures, Collections are designed to handle larger data sets more efficiently.
**Convenience:** Collections can be used not only for arrays but also for Eloquent query results. If you fetch records from the database, Eloquent returns a Collection, which means you can immediately use all the collection methods on the result.

**Using Collections for Database Queries**

Instead of using plain arrays to manipulate query results, you can directly use Collections on Eloquent query results:

    $users = User::where('active', true)
        ->get() // Returns a collection
        ->filter(fn($user) => $user->age > 30)
        ->sortBy('age')
        ->pluck('name');


In this example, the ->get() method retrieves a collection of active users. Then, we filter, sort, and pluck the names from the collection using chainable methods.
