
# 1. Basic Model Attributes
   ## Accessing Attributes :  involves retrieving the values of attributes from an Eloquent model instance. This is a fundamental aspect of working with Eloquent models, allowing  you to interact with the data stored in your database.
      use App\Models\User;
      $user = User::find(1);
      echo $user->name;

## Mass Assignment: Mass assignment is when you send an array to the model creation, basically setting a bunch of fields on the model in a single go, rather than one by one, something like:
$user = new User(request()->all());

(This is instead of explicitly setting each value on the model separately.)

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


