
# 1. Basic Model Attributes
   ## Accessing Attributes :  involves retrieving the values of attributes from an Eloquent model instance. This is a fundamental aspect of working with Eloquent models, allowing  you to interact with the data stored in your database.
      use App\Models\User;
      $user = User::find(1);
      echo $user->name;

## Mass Assignment: Mass assignment is when you send an array to the model creation, basically setting a bunch of fields on the model in a single go, rather than one by one, something like:

     $user = new User(request()->all());  // This is instead of explicitly setting each value on the model separately.

You can use fillable to protect which fields you want this to actually allow for updating.
You can also block all fields from being mass-assignable by doing this:

    protected $guarded = ['*'];

Let's say in your user table you have a field that is user_type and that can have values of user / admin
Obviously, you don't want users to be able to update this value. In theory, if you used the above code, someone could inject into a form a new field for user_type and send 'admin' along with the other form data, and easily switch their account to an admin account... bad news.
By adding:

    $fillable = ['name', 'password', 'email'];

You are ensuring that only those values can be updated using mass assignment
To be able to update the user_type value, you need to explicitly set it on the model and save it, like this:

      $user->user_type = 'admin';
      $user->save();
# 2. Accessors and Mutators
  ## Accessors : Accessors allow you to transform Eloquent attribute values when you retrieve them. This is useful when you need to format or manipulate data before presenting it.
  
Let’s say you have a User model and you want to ensure that the name is always capitalized when retrieved. You can define an accessor to achieve this.

     class User extends Model
    {
    // Define an accessor for the 'name' attribute
    public function getNameAttribute($value)
    {
        return ucfirst($value); // Capitalize the first letter
    }
    }

    $user = User::find(1);
    echo $user->name; // Outputs the 'name' attribute with the first letter capitalized
    
  ## Mutators : Mutators allow you to transform Eloquent attribute values when you set them. This is useful when you need to format or manipulate data before saving it to the database.
  
Consider a scenario where you want to hash a user's password before saving it to the database. You can define a mutator to handle this.

     class User extends Model
    {
     // Define a mutator for the 'password' attribute
    public function setPasswordAttribute($value)
    {
        $this->attributes['password'] = bcrypt($value); // Hash the password
    }
    }

    $user = new User();
   $user->password = '123456'; // The password will be hashed before saving
   $user->save();

# 3. Appending Custom Attributes

  you can append custom attributes to the model's array form using the $appends property. This allows you to include additional computed or derived data alongside the actual database columns when serializing the model instance.

  Let's say you have a User model with separate first_name and last_name attributes, and you want to retrieve the user's full name as a single attribute.

     class User extends Model
    {
     protected $appends = ['full_name'];

    public function getFullNameAttribute()
    {
        return $this->first_name . ' ' . $this->last_name;
    }
    }

    $user = User::find(1);
    echo $user->full_name; // Outputs the full name of the user

    
# 4. Attribute Serialization

 Attribute serialization allows you to control which attributes are included when converting a model instance to an array or JSON format. This is useful for hiding sensitive information or reducing the size of JSON responses.
 
Let's say you have a User model with sensitive attributes like password and remember_token, which you don't want to expose in JSON response

     class User extends Model
    {
    protected $hidden = ['password', 'remember_token'];
    }

   **Attribute serialization offers several benefits:**
   
  **Security**: You can protect sensitive data, such as passwords or tokens, from being unintentionally exposed in JSON responses.
   
  **Performance**: By reducing the size of JSON responses, you can improve the efficiency of data transmission between your application and clients.
   
   **Control**: You have fine-grained control over what data is exposed externally, ensuring compliance with security and privacy requirements.
   

   # 5. Attribute Casting:
   
   Attribute casting in Laravel allows you to specify how Eloquent attributes should be converted to and from native PHP types when interacting with the database. This feature simplifies data handling by automatically transforming attribute values into appropriate data types.

     class User extends Model
    {
      protected $casts = [
              'is_admin' => 'boolean',
              'age' => 'integer',
              'salary' => 'float',
              'settings' => 'array',
              'joined_at' => 'datetime',
          ];
    }
       
# 6. Custom Attribute Casts:
Attribute casting allows you to specify how attribute values should be transformed when interacting with the database. While Laravel provides built-in casts for common data types, you can also define custom attribute casts to handle specialized data transformations.

Let's create a custom attribute cast that converts attribute values to uppercase when setting them and to lowercase when retrieving them.

First, define a class that implements the CastsAttributes interface:

      use Illuminate\Contracts\Database\Eloquent\CastsAttributes;

      class AsUppercase implements CastsAttributes
      {
          public function get($model, string $key, $value, array $attributes)
          {
              return strtolower($value); // Convert value to lowercase when retrieving
          }
      
          public function set($model, string $key, $value, array $attributes)
          {
              return strtoupper($value); // Convert value to uppercase when setting
          }
      }
      
Next, use the custom cast class in your Eloquent model:

      class User extends Model
      {
          protected $casts = [
              'name' => AsUppercase::class,
          ];
      }
**Explanation:**

     1. The AsUppercase class implements the CastsAttributes interface, which requires defining get and set methods.
     
     2. In the get method, the value retrieved from the database ($value) is transformed to lowercase before being returned.
     
     3. In the set method, the value being set ($value) is transformed to uppercase before being assigned to the attribute.
     
     4. The $casts property in the User model specifies that the name attribute should be cast using the AsUppercase class.
     
# 7. Using Accessors for Relationships:
You can define accessors to retrieve attributes from related models. This allows you to access related model data as if it were a native attribute of the current model, enhancing the flexibility and readability of your code.

Let's create an accessor in a Post model that retrieves the name of the post's author through a belongsTo relationship with the User model

         class Post extends Model
         {
             public function author()
             {
                 return $this->belongsTo(User::class);
             }
         
             public function getAuthorNameAttribute()
             {
                 return $this->author->name;
             }
         }

# 8. Complex Accessors and Mutators :
Complex accessors and mutators allow you to define attribute transformations that involve additional logic or calculations. This is particularly useful for computing derived attributes based on related data or performing aggregations.
Let's create a total accessor in an Order model that computes the total price of the order based on its items.

         class Order extends Model
         {
             protected $appends = ['total'];
         
             public function getTotalAttribute()
             {
                 return $this->items->sum(function ($item) {
                     return $item->quantity * $item->price; // Calculate item subtotal
                 });
             }
         }

**Benefits of Complex Accessors and Mutators** 
Complex accessors and mutators offer several advantages:

**Data Aggregation**: You can aggregate data across related models or perform complex calculations to derive attribute values.

**Encapsulation**: Logic for computing derived attributes is encapsulated within the model, promoting cleaner and more maintainable code.

**Consistency**: By defining attribute transformations centrally, you ensure consistent data presentation throughout your application.

# 9. Scope:
Scopes in Laravel Eloquent models are predefined methods that help you encapsulate common query logic. They make it easier to reuse query constraints throughout your application without repeating code.

Let's create a scope in a Task model to filter tasks based on their status, such as completed or incomplete.

         use Illuminate\Database\Eloquent\Model;
         use Illuminate\Database\Eloquent\Builder;
         
         class Task extends Model
         {
             public function scopeCompleted($query)
             {
                 return $query->where('status', 'completed');
             }
         
             public function scopeIncomplete($query)
             {
                 return $query->where('status', 'incomplete');
             }
         }

Here’s how you can use these scopes in a controller to retrieve tasks:

         use App\Models\Task;
         
         class TaskController extends Controller
         {
             public function index()
             {
                 // Retrieve completed tasks
                 $completedTasks = Task::completed()->get();
         
                 // Retrieve incomplete tasks
                 $incompleteTasks = Task::incomplete()->get();
         
                 // Return tasks to view or process further
                 return view('tasks.index', compact('completedTasks', 'incompleteTasks'));
             }
         }


# 10. Global Scopes:

Global scopes in Laravel Eloquent models allow you to apply constraints to all queries for a given model. They are automatically applied whenever a query is executed on that model, making them useful for enforcing conditions consistently without explicitly adding them to every query.

Let's create a global scope in a User model to ensure that only active users are retrieved by default.

         use Illuminate\Database\Eloquent\Model;
         use Illuminate\Database\Eloquent\Builder;
         
         class User extends Model
         {
             protected static function boot()
             {
                 parent::boot();
         
                 static::addGlobalScope('active', function (Builder $builder) {
                     $builder->where('active', true);
                 });
             }
         }

The boot method is overridden in the User model to add a global scope.

You can disable a global scope temporarily using the withoutGlobalScope method:

         $users = User::withoutGlobalScope('active')->get();

Here’s how you can use a global scope in a controller to retrieve users:

      
      use App\Models\User;
      
      class UserController extends Controller
      {
          public function index()
          {
              // Retrieve all active users (global scope applied)
              $users = User::all();
      
              // Retrieve all users (ignoring global scope)
              $allUsers = User::withoutGlobalScope('active')->get();
      
              // Return users to view or process further
              return view('users.index', compact('users', 'allUsers'));
          }
      }


