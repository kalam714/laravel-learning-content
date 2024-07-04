
# 1. Basic Model Attributes
   ## Accessing Attributes :  involves retrieving the values of attributes from an Eloquent model instance. This is a fundamental aspect of working with Eloquent models, allowing  you to interact with the data stored in your database.
      use App\Models\User;
      $user = User::find(1);
      echo $user->name;

