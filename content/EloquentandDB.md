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

