# Collections

collections are a powerful and flexible way to work with arrays of data. They are based on PHP’s native array functions but provide an expressive, object-oriented syntax.

Using the **User** model with various collection methods to work with user data in Laravel. We’ll assume you have a users table with fields like name, email, age, and created_at.

    // Fetching All Users
    use App\Models\User;
    $users = User::all(); // Fetches all users from the database
    
    //filter user based on age
    
    $filteredUsers = $users->filter(function ($user) {
        return $user->age > 25;
    });
    
    //Map Method to Extract Only Names
    $userNames = $users->map(function ($user) {
        return $user->name;
    });
    // ["kalam", "imran", "nahid"]
    
    $sortedUsers = $users->sortBy('age');
    // Users sorted by age in ascending order
    
    //Find Users with Specific Criteria
    $containsEmail = $users->contains('email', 'john@example.com'); // Returns true if exists
    
    $groupedByAge = $users->groupBy('age');
    // Grouped users by their age, e.g., [25 => [...users with age 25], 30 => [...users with age 30]]
    
    //Partition users into two groups, one with age over 30 and one with age 30 or below:
    
    list($olderUsers, $youngerUsers) = $users->partition(function ($user) {
        return $user->age > 30;
    });
    // $olderUsers will contain users with age > 30, and $youngerUsers will contain others.
    
    $userCount = $users->count();
    // e.g., 50 users in total
    
    $totalAge = $users->sum('age');
    // Sum of all users' ages
    
    //Let’s say you want to reduce the collection to a single value, such as the total age of all users:
    
    $totalAge = $users->reduce(function ($carry, $user) {
        return $carry + $user->age;
    }, 0);
    // $totalAge will be the sum of ages of all users
    
    $userChunks = $users->chunk(10);
    // Split users into batches of 10
    
    //Combining Keys and Values
    
    $userIds = $users->pluck('id');
    $userNames = $users->pluck('name');
    
    $combined = $userIds->combine($userNames);
    // Example result: [1 => 'John', 2 => 'Jane', 3 => 'Tom']
    
    //key-value pair
    $usersById = $users->keyBy('id');
    // Now you can access users by their id, e.g., $usersById[1] will return the user with id 1
    
    //Filtering Users by Multiple Conditions
    
    $filteredUsers = $users->filter(function ($user) {
        return $user->age > 25 && strpos($user->email, '@example.com') !== false;
    });
    // Get users older than 25 with emails from @example.com
    
    //Mapping Users to Custom Data
    
    $userSummaries = $users->map(function ($user) {
        return [
            'name' => $user->name,
            'age' => $user->age,
            'status' => $user->age > 30 ? 'Senior' : 'Junior'
        ];
    });
    // Create a list of users with a custom 'status' based on their age
    
    //Sorting Users by Multiple Criteria
    
    $sortedUsers = $users->sortBy([
        ['age', 'desc'],
        ['name', 'asc']
    ]);
    // Sort users by age in descending order and name in ascending order
    
    //Chaining Methods Together
    
    $userNames = $users->filter(function ($user) {
        return $user->age > 25;
    })->sortByDesc('age')->pluck('name');
    // First filter users older than 25, then sort by age in descending order, and finally pluck the names
    
    //Using each to Perform an Action on Each User
    
    $users->each(function ($user) {
        // Do something with each user, for example, update their status
        $user->status = 'Active';
        $user->save();
    });
    
    //Flattening a Nested Collection
    
    $nestedUsers = collect([
        collect([['name' => 'John'], ['name' => 'Jane']]),
        collect([['name' => 'Tom'], ['name' => 'Alice']]),
    ]);
    
    $flattenedUsers = $nestedUsers->flatten();
    // Flattened users: [['name' => 'John'], ['name' => 'Jane'], ['name' => 'Tom'], ['name' => 'Alice']]







