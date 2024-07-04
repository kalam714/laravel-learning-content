
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

   Attribute serialization offers several benefits:
   
   Security: You can protect sensitive data, such as passwords or tokens, from being unintentionally exposed in JSON responses.
   
   Performance: By reducing the size of JSON responses, you can improve the efficiency of data transmission between your application and clients.
   
   Control: You have fine-grained control over what data is exposed externally, ensuring compliance with security and privacy requirements.
       
     
