# Eager Loading with Constraints

Eager loading optimizes queries by reducing the number of database calls. You can apply constraints to eager-loaded relationships.

    $users = User::with(['posts' => function ($query) {
        $query->where('status', 'published');
    }])->get();

#  Custom Relationship Methods
Define custom methods for relationships to extend functionality.

    public function activePosts()
    {
        return $this->hasMany(Post::class)->where('status', 'active');
    }
    
    // Usage
    $activePosts = $user->activePosts()->get();

# Polymorphic Relationships with Multiple Levels

Polymorphic relationships in Laravel are a type of Eloquent relationship that allows a model to belong to more than one other model on a single association. This is achieved by using a shared table structure, which makes it reusable and flexible for multiple related models.
**Scenario**
You have two entities:
  + Posts
  + Videos
    
Both entities can have comments, but instead of creating separate tables for post comments and video comments, you can use a single comments table to store comments for both.

**Setting Up Polymorphic Relationships**

**Database Schema** : You need to create a comments table with fields to identify the related model and its record.

    Schema::create('comments', function (Blueprint $table) {
        $table->id();
        $table->text('content');
        $table->morphs('commentable'); // Creates `commentable_id` and `commentable_type`
        $table->timestamps();
    });

The morphs function creates two fields:

  + commentable_id: The ID of the related record.
  + commentable_type: The class name of the related model (e.g., Post, Video).

**Model Definitions**

    //Comment model
    
    class Comment extends Model
    {
        public function commentable()
        {
            return $this->morphTo();
        }
    }

    //Post model
    
    class Post extends Model
    {
        public function comments()
        {
            return $this->morphMany(Comment::class, 'commentable');
        }
    }

    //Video model

    class Video extends Model
    {
        public function comments()
        {
            return $this->morphMany(Comment::class, 'commentable');
        }
    }

**Usage**

    //To add a comment to a post:

    $post = Post::find(1);
    $post->comments()->create([
        'content' => 'This is a comment on a post.',
    ]);

    //To add a comment to a video:

    $video = Video::find(1);
    $video->comments()->create([
        'content' => 'This is a comment on a video.',
    ]);

    //To get all comments for a specific post:

    $post = Post::find(1);
    $comments = $post->comments; // Returns a collection of comments

    //To get all comments for a specific video:

    $video = Video::find(1);
    $comments = $video->comments;

    //Accessing the Related Model from a Comment

    $comment = Comment::find(1);
    $relatedModel = $comment->commentable; // Returns either a Post or Video model instance


**Behind the Scenes**

  + Laravel uses the commentable_id and commentable_type fields to determine which model the comment belongs to.
  + The commentable_type stores the fully qualified class name (App\Models\Post or App\Models\Video).
  + When accessing the commentable relationship, Laravel automatically queries the correct model based on these fields.

**Advantages of Polymorphic Relationships**

  + **Reusability**: The comments table can handle comments for any number of models (e.g., posts, videos, products).
  + **Scalability**: Easily extendable to other models without creating additional tables.
  + **Simplicity**: Centralizes the handling of related data in one table and relationship.


# Subqueries and Raw Expressions

Subqueries and raw expressions are powerful tools in Laravel for crafting advanced database queries. They allow for more complex operations and efficient data retrieval by utilizing SQL features directly within Eloquent.

**Subqueries **

Subqueries  allow you to fetch data from related tables or perform complex calculations in a single query. This is useful when you need additional computed or related data within your query.


    $users = User::select('id', 'name', function ($query) {
        $query->select('title') // Select the `title` column
            ->from('posts') // From the `posts` table
            ->whereColumn('posts.user_id', 'users.id') // Match `posts.user_id` with `users.id`
            ->orderByDesc('created_at') // Order posts by latest creation date
            ->limit(1); // Fetch only the latest post
    })->get();

**Generated SQL**

    SELECT 
        id, 
        name, 
        (SELECT title 
         FROM posts 
         WHERE posts.user_id = users.id 
         ORDER BY created_at DESC 
         LIMIT 1) as title 
    FROM users;

**Raw Expressions**
Raw expressions allow you to write SQL expressions directly, bypassing Eloquent's abstraction. This is useful for:

  + Aggregations (e.g., COUNT, SUM, AVG)
  + Custom calculations
  + Grouping and advanced filtering

    $users = User::selectRaw('COUNT(*) as user_count, status')
        ->groupBy('status')
        ->get();

**Generated SQL**

    SELECT 
        COUNT(*) as user_count, 
        status 
    FROM users 
    GROUP BY status;

**Comparing Subqueries and Raw Expressions**

  
  | Feature      | Subqueries                         | Raw Expressions               |
  |--------------|------------------------------------|-------------------------------|
  | **Purpose**  | Embed a query within another query | Directly use SQL syntax in Eloquent |
  | **Complexity** | Moderate to High                 | Low to Moderate               |
  | **Use Cases** | Fetch related or computed data    | Aggregations, custom calculations |
  | **Example**   | Fetch latest post titles for users | Count users by status         |


**Combining Subqueries and Raw Expressions**

You can combine subqueries and raw expressions for even more complex queries. For example:


    $users = User::select('id', 'name')
        ->addSelect(['latest_post' => function ($query) {
            $query->selectRaw('MAX(created_at)')
                ->from('posts')
                ->whereColumn('posts.user_id', 'users.id');
        }])
        ->get();


**Subquery with selectRaw:**
  + Retrieves the latest created_at date for each user's posts.
**Main Query:**
 + Fetches id and name along with the result of the subquery (latest_post).

**Generated SQL**

    SELECT 
        id, 
        name, 
        (SELECT MAX(created_at) 
         FROM posts 
         WHERE posts.user_id = users.id) as latest_post 
    FROM users;

**Performance**: 

  + Avoid overusing subqueries; they can impact performance for large datasets.
  + Consider database indexes on fields used in subqueries and groupBy.
    
**When to Use Raw Expressions:** Use raw expressions for database-specific functions or advanced aggregations that Eloquent doesn't natively support.

**Debugging:** Use toSql() to view the generated SQL and ensure correctness.

    $query = User::selectRaw('COUNT(*) as user_count, status')
        ->groupBy('status');
    dd($query->toSql());

# Pivot Table Operations

Laravel allows you to work with pivot tables when dealing with many-to-many relationships. A pivot table is an intermediary table that stores the relationships between two other tables. You can also store additional information in the pivot table, such as timestamps or custom columns, to enrich the relationship.

**Scenario**

You have a users table and a roles table, with a pivot table role_user to handle the many-to-many relationship. The pivot table includes:

  + user_id: The user associated with the role.
  + role_id: The role assigned to the user.
  + created_at: Timestamp for when the role was assigned.
  + expires_at: A custom column indicating when the role expires.

**Defining the Relationship**

    // User model
    public function roles()
    {
        return $this->belongsToMany(Role::class)
                    ->withPivot('created_at', 'expires_at'); // Specify extra pivot columns
    }


**Explanation**

  + belongsToMany: Establishes the many-to-many relationship between User and Role.
  + withPivot: Ensures Laravel includes the specified pivot columns (created_at, expires_at) when interacting with the relationship.


**Pivot Table Operations**

| Operation          | Method                  | Description                                                         |
|---------------------|-------------------------|---------------------------------------------------------------------|
| **Attach a role**   | `attach()`             | Adds a role with optional extra data to the pivot table.            |
| **Update pivot data** | `updateExistingPivot()` | Updates an existing row in the pivot table.                         |
| **Detach a role**   | `detach()`             | Removes the relationship between the user and role.                 |
| **Synchronize roles** | `sync()`               | Syncs the pivot table to match the provided array of data.           |
| **Access pivot data** | `$role->pivot->column` | Access extra data from the pivot table for a relationship.          |

**Attaching with Extra Data**

    $user = User::find(1);
    $roleId = 2;
    
    $user->roles()->attach($roleId, [
        'expires_at' => now()->addDays(30) // Set custom data for the pivot column
    ]);


roles()->attach($roleId, $attributes):

  + $roleId: The ID of the role being attached to the user.
  + $attributes: An associative array of additional pivot table data (e.g., expires_at).
    
Generated SQL:

    INSERT INTO `role_user` (`user_id`, `role_id`, `expires_at`, `created_at`) 
    VALUES (1, 2, '2024-12-08 00:00:00', '2024-12-07 00:00:00');

**Retrieving Extra Pivot Data**

    $user = User::find(1);
    
    foreach ($user->roles as $role) {
        echo $role->pivot->expires_at; // Access the `expires_at` column from the pivot table
    }
    
$role->pivot: The pivot table row that connects the user and the role.

**Updating Pivot Data**

    $user = User::find(1);
    $user->roles()->updateExistingPivot($roleId, [
        'expires_at' => now()->addDays(60) // Update the expiration date
    ]);

**Generated SQL:**

    UPDATE `role_user` 
    SET `expires_at` = '2024-02-05 00:00:00' 
    WHERE `user_id` = 1 AND `role_id` = 2;

**Removing Relationships**

    $user = User::find(1);
    $user->roles()->detach($roleId); // Removes the row from the pivot table

    $user->roles()->detach(); // Removes all rows related to the user


**Synchronizing Relationships**

The sync() method can update the pivot table with an array of role IDs and their extra data:

    $user->roles()->sync([
        1 => ['expires_at' => now()->addDays(30)],
        2 => ['expires_at' => now()->addDays(60)],
    ]);

Existing rows are updated, new rows are inserted, and any roles not in the array are removed.

**What Happens Without a Pivot Table?**

If you attempt to use belongsToMany or perform operations like attach() or sync() without the pivot table:
Laravel will throw an error because the table does not exist in the database.

    SQLSTATE[42S02]: Base table or view not found: 1146 Table 'your_database.role_user' doesn't exist


# Using Transactions for Complex Database Operations

In Laravel, transactions make sure that a group of database changes are done completely or not at all. This is useful when multiple things need to happen together, and it would cause problems if only some of them succeed.

**Simple Explanation with an Example**

Imagine you're shopping online, and you're buying something:

+ Money should be taken from your account.
+ The item should be marked as sold in the store.

    
Both of these steps must happen together. If the money is taken but the item isn't marked as sold (or vice versa), something is wrong. This is where a transaction helps.

**How Transactions Work in Laravel**

+ Laravel puts all related database changes in a "box" (a transaction).
+ If all changes in the box succeed, Laravel saves the changes.
+ If something goes wrong, Laravel throws away the box (rolls back), and the database stays the same as before.

**Scenario: Placing an Order**

When a customer places an order, multiple things need to happen:

+ Order Creation: Save the order details (like total amount, customer info, etc.) in the database.
+ Inventory Update: Deduct the ordered items from the inventory.
+ Payment Processing: Record the payment transaction.


    
All of these steps must happen together. If any step fails (e.g., inventory is not updated, or payment fails), none of the changes should be saved.

Here’s how a transaction ensures everything succeeds or nothing happens:

        DB::transaction(function () {
            // Step 1: Create the order
            $order = Order::create([
                'customer_id' => 123,        // Customer placing the order
                'total_amount' => 5000,     // Total cost of the order
                'status' => 'pending',      // Initial status of the order
            ]);
        
            // Step 2: Deduct items from inventory
            foreach ($order->items as $item) {
                $product = Product::find($item['product_id']);
                if ($product->stock < $item['quantity']) {
                    throw new \Exception("Insufficient stock for product ID: {$item['product_id']}");
                }
                $product->decrement('stock', $item['quantity']); // Deduct the stock
            }
        
            // Step 3: Record the payment
            Payment::create([
                'order_id' => $order->id,
                'amount' => $order->total_amount,
                'status' => 'successful', // Assume payment is successful for this example
            ]);
        
            // Final Step: Update order status to "completed"
            $order->update(['status' => 'completed']);
        });

**What Happens Here**

**Transaction Begins**: All operations are held temporarily in a "box" (the transaction).
**Order Creation**: The order is saved with pending status.
**Inventory Update**: Each product’s stock is checked and updated.
If stock is insufficient for any product, an exception is thrown, and the transaction stops.
**Payment Recording**: Payment details are saved in the database.
**Commit or Rollback**: If all steps succeed, the transaction is committed (saved permanently).
If any step fails (e.g., insufficient stock), the transaction rolls back (cancels everything).

**Why Transactions Are Important Here**

**Without a Transaction**

If the order is created and payment is recorded, but the inventory update fails, the system is left in an inconsistent state:
The customer sees their order as successful.
Inventory does not reflect the stock properly.

**With a Transaction**

If any step fails, nothing is saved.
The database stays clean and accurate, avoiding customer confusion or stock issues.

**Manual Transaction Control**

If you need more control, you can manually start, commit, or roll back transactions:

        DB::beginTransaction();
        
        try {
            $user = User::create([...]);
            $user->posts()->create([...]);
        
            // Commit transaction
            DB::commit();
        } catch (\Exception $e) {
            // Rollback transaction if an error occurs
            DB::rollBack();
        
            // Handle exception (e.g., log error, rethrow, etc.)
            throw $e;
        }


**DB::beginTransaction()**: Starts the transaction manually.
**DB::commit()**: Commits the transaction to save changes.
**DB::rollBack()**: Rolls back the transaction in case of errors.

**Transaction Summary**

| Feature          | Detail                                                                 |
|-------------------|-----------------------------------------------------------------------|
| **What it does**  | Ensures multiple operations either succeed together or fail together (rollback). |
| **When to use**   | For multi-step operations where partial completion would leave the database inconsistent. |
| **Automatic**     | Rolls back automatically if an exception occurs in the callback function. |
| **Manual**        | Use `beginTransaction`, `commit`, and `rollBack` for greater control. |
| **Best Practices** | Keep transactions short, handle exceptions properly, and avoid unnecessary nesting. |


# Model Observers 

A Model Observer is a clean and centralized way to handle various events (e.g., created, updated, deleted) that occur on an Eloquent model. Instead of scattering event-handling logic throughout your application, Observers let you define all event-related logic in a single class.

**When to Use Observers?**
To run logic automatically after specific events, such as:
+ Sending an email when a user is registered.
+ Logging changes when a record is updated.
+ Cleaning up related data when a record is deleted.

**Key Events an Observer Can Handle**

**creating**: Before a record is created.
**created**: After a record is created.
**updating**: Before a record is updated.
**updated**: After a record is updated.
**deleting**: Before a record is deleted.
**deleted**: After a record is deleted.
**retrieved**: When a record is retrieved from the database.
**restoring**: Before a soft-deleted record is restored.
**restored**: After a soft-deleted record is restored.

You can generate an observer using the Artisan command:

        php artisan make:observer UserObserver --model=User

**Generated Observer Example**

        namespace App\Observers;
        
        use App\Models\User;
        
        class UserObserver
        {
            public function created(User $user)
            {
                // Logic after a user is created
                \Log::info("A new user was created: {$user->email}");
            }
        
            public function updated(User $user)
            {
                // Logic after a user is updated
                \Log::info("User updated: {$user->email}");
            }
        
            public function deleted(User $user)
            {
                // Logic after a user is deleted
                \Log::info("User deleted: {$user->email}");
            }
        }


**Use Cases**

        //Sending a Welcome Email After User Registration
        
        public function created(User $user)
        {
            \Mail::to($user->email)->send(new WelcomeEmail($user));
        }
        
        //Auto-Generating a Username
        
        public function creating(User $user)
        {
            $user->username = strtolower(str_replace(' ', '_', $user->name)) . rand(1000, 9999);
        }
        
        //Logging Updates
        
        public function updated(User $user)
        {
            \Log::info("User {$user->id} has been updated at {$user->updated_at}");
        }


**Advantages of Using Observers**

**Centralized Logic**:All event-related logic for a model is in one place, making it easier to manage and maintain.
**Clean Codebase**:Reduces repetitive code across controllers or services by abstracting event handling into Observers.
**Decoupling**:Keeps models and controllers focused on their primary responsibilities.
**Reusability**: You can reuse Observers across multiple models if the logic is generic.

**Observer vs Event Listeners**

**Observers**: Best for handling model-specific events, such as created or updated.
**Event Listeners**: Ideal for global events unrelated to a specific model (e.g., user logging in or out).

**Summary**

| Feature         | Details                                                                 |
|------------------|------------------------------------------------------------------------|
| **Purpose**      | Handle model events (created, updated, etc.) in a centralized way.     |
| **Usage**        | Define logic in an Observer class and register it to the model.        |
| **Best Practices** | Use Observers for model-specific event handling to keep your code clean. |


# Advanced Casting and Accessors 

Casting and accessors are powerful features in Laravel that help you manipulate how data is retrieved from or stored in the database.

**Custom Casting**

Custom casting allows you to transform how specific attributes of your model are stored and retrieved from the database. Laravel’s default $casts handles common types like json, array, or datetime, but custom casting lets you define your own logic.

**How Custom Casting Works**

You create a class that implements the CastsAttributes interface.

Define two methods:

+ get: How to transform the attribute when retrieved from the database.
+ set: How to transform the attribute before saving to the database.

**Example: Casting JSON Data**

**Scenario**: You have a preferences column in your database that stores JSON data. You want to store it as a JSON string in the database but work with it as an array in your code.

**Custom Caster Implementation:**

        use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
        
        class JsonCast implements CastsAttributes
        {
            // Transform data when retrieving from the database
            public function get($model, $key, $value, $attributes)
            {
                return json_decode($value, true); // Decode JSON string to array
            }
        
            // Transform data before saving to the database
            public function set($model, $key, $value, $attributes)
            {
                return json_encode($value); // Encode array to JSON string
            }
        }

**Using the Custom Caster in the Model:**


        class User extends Model
        {
            protected $casts = [
                'preferences' => JsonCast::class, // Apply custom cast for 'preferences'
            ];
        }
        
        //useage
        
        $user = User::find(1);
        
        // Accessing 'preferences' (JSON decoded as an array)
        $preferences = $user->preferences; // ['theme' => 'dark', 'language' => 'en']
        
        // Modifying 'preferences'
        $user->preferences = ['theme' => 'light', 'language' => 'fr'];
        $user->save(); // Saved as JSON string in the database


**Complex Accessors**

Accessors allow you to define virtual attributes for a model. These attributes don’t exist in the database but are dynamically created based on the model’s actual data.

**How Accessors Work**

+ Create a public method in the model with the name pattern get{AttributeName}Attribute.
+ Use it to dynamically compute and return the value of the virtual attribute.

**Example: Combining First and Last Name**

**Scenario**: You want a full_name attribute for a user, but the database only stores first_name and last_name.

        class User extends Model
        {
            public function getFullNameAttribute()
            {
                return "{$this->first_name} {$this->last_name}";
            }
        }
        
        echo $user->full_name;
        
        //Nested Accessor with Transformation
        
        public function getAddressAttribute()
        {
            return "{$this->street}, {$this->city}, {$this->state}, {$this->zip_code}";
        }
        
        echo $user->address; 



**Key Differences Between Casting and Accessors**

| Feature          | Custom Casting                                   | Accessors                                    |
|-------------------|--------------------------------------------------|---------------------------------------------|
| **Purpose**       | Define how attributes are saved/retrieved in the DB. | Create virtual attributes dynamically.      |
| **Storage Impact** | Affects how data is stored in the database.      | Does not affect storage, only representation. |
| **Methods Used**  | `get` and `set` methods in the caster class.      | `get{AttributeName}Attribute` method in the model. |
| **Example**       | JSON transformations for database columns.       | Concatenating first and last name.          |

**When to Use**

+ Use custom casting for attributes that need specific storage and retrieval transformations (e.g., encrypting data, converting JSON, etc.).
+ Use accessors for creating attributes derived from other fields or relationships.


**Query Scopes**

Query scopes in Laravel are a way to encapsulate common query logic so it can be reused across your application. Scopes come in two types:

**Global Scopes**: Automatically apply to all queries on a model unless explicitly removed.
**Local Scopes**: Explicitly applied to queries when needed.

**Global Scopes**

Automatically modify the query logic for a model every time it is queried. They are useful when you want a consistent filter across all queries for a model (e.g., only show active users).

**How to Implement a Global Scope**

**Create a Scope Class**: A scope class implements the Illuminate\Database\Eloquent\Scope interface and defines an apply method, where you modify the query.

        namespace App\Scopes;
        
        use Illuminate\Database\Eloquent\Builder;
        use Illuminate\Database\Eloquent\Model;
        use Illuminate\Database\Eloquent\Scope;
        
        class ActiveScope implements Scope
        {
            // Modify the query to include only active records
            public function apply(Builder $builder, Model $model)
            {
                $builder->where('active', true);
            }
        }

**Apply the Scope to a Model**: You register the scope in the booted method of the model.

        namespace App\Models;
        
        use App\Scopes\ActiveScope;
        use Illuminate\Database\Eloquent\Model;
        
        class User extends Model
        {
            protected static function booted()
            {
                // Add the global scope
                static::addGlobalScope(new ActiveScope);
            }
        }

**Query with Global Scope Applied**: By default, all queries for this model will now include the global scope.

        $users = User::all(); // Only retrieves users where 'active' = true

**Remove a Global Scope**: If you need to bypass the global scope for specific queries, you can use the withoutGlobalScope method.

        $users = User::withoutGlobalScope(ActiveScope::class)->get(); // Retrieves all users

**Global Scope with Parameters**:You can make your global scope dynamic by passing parameters.

        class ActiveScope implements Scope
        {
            protected $status;
        
            public function __construct($status)
            {
                $this->status = $status;
            }
        
            public function apply(Builder $builder, Model $model)
            {
                $builder->where('active', $this->status);
            }
        }
        
        // Applying the scope
        static::addGlobalScope(new ActiveScope(true));


**Local Scopes**

Local Scopes are reusable query constraints defined directly in the model. Unlike global scopes, they are not applied automatically and must be explicitly invoked.

**How to Implement a Local Scope**

**Define a Scope Method**: Create a method in your model with the scope prefix and the name of the scope. It accepts the query builder as the first argument.

        namespace App\Models;
        
        use Illuminate\Database\Eloquent\Model;
        
        class Post extends Model
        {
            // Define a local scope
            public function scopePopular($query)
            {
                return $query->where('popularity', '>', 100);
            }
        
            // Another local scope
            public function scopeRecent($query)
            {
                return $query->orderBy('created_at', 'desc');
            }
        }

**Use the Scope in Queries**: When querying the model, call the scope as a method without the scope prefix.

        // Retrieve popular posts
        $popularPosts = Post::popular()->get();
        
        // Combine multiple local scopes
        $recentPopularPosts = Post::popular()->recent()->get();

** Local Scope with Parameters**: You can pass arguments to local scopes for more flexibility.

        public function scopePopular($query, $threshold = 100)
        {
            return $query->where('popularity', '>', $threshold);
        }
        
        // Usage
        $popularPosts = Post::popular(200)->get(); // Popular posts with a threshold of 200

**Summary**

Query scopes make your queries cleaner, more reusable, and easier to maintain. Use global scopes for consistent, automatic filtering and local scopes for optional constraints that enhance query flexibility.
