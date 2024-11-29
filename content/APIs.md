# Eloquent: API Resources
## provide a convenient and standardized way to transform your Eloquent models into JSON responses for APIs. They help you control the structure and format of the API output while adhering to best practices like separating the database logic from presentation layers.Ensure a clean separation between your data layer and your API's presentation, improving maintainability and consistency.

    php artisan make:resource PostResource
  This generates a PostResource class in the App\Http\Resources directory.
  
The toArray method of the resource class allows you to define how the model's attributes will be transformed:

    namespace App\Http\Resources;
    use Illuminate\Http\Resources\Json\JsonResource;
    
    class PostResource extends JsonResource
    {
        /**
         * Transform the resource into an array.
         */
        public function toArray($request)
        {
            return [
                'id' => $this->id,
                'title' => $this->title,
                'content' => $this->content,
                'author' => $this->author->name,
                'published_at' => $this->created_at->format('Y-m-d H:i:s'),
            ];
        }
    }

To return a transformed response, use the PostResource in your controller:

    use App\Http\Resources\PostResource;
    use App\Models\Post;
    
    class PostController extends Controller
    {
        public function show($id)
        {
            $post = Post::findOrFail($id);
    
            return new PostResource($post);
        }
    }

For multiple records, Laravel provides a resource collection. You can use the collection method or create a dedicated collection resource:

    php artisan make:resource PostCollection

Alternatively, use the PostResource::collection method:

    public function index()
    {
        $posts = Post::all();
    
        return PostResource::collection($posts);
    }

You can include additional metadata in the response using the with method:

    public function with($request)
    {
        return [
            'meta' => [
                'version' => '1.0.0',
                'author' => 'Your Name',
            ],
        ];
    }

You can include fields conditionally using when or mergeWhen:

    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'content' => $this->content,
            'is_published' => $this->is_published,
            'published_at' => $this->when($this->is_published, $this->published_at),
            'extra_data' => $this->mergeWhen(auth()->check(), [
                'edit_link' => route('posts.edit', $this->id),
            ]),
        ];
    }
    
You can use mergeWhen to include additional attributes in the resource output based on specific conditions.

    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'content' => $this->content,
            'is_published' => $this->is_published,
            
            // Merge additional data only when the user is authenticated
            $this->mergeWhen(auth()->check(), [
                'edit_url' => route('posts.edit', $this->id),
                'delete_url' => route('posts.destroy', $this->id),
            ]),
    
            // Merge based on a custom logic
            $this->mergeWhen($this->likes > 100, [
                'is_trending' => true,
            ]),
        ];
    }

You can use whenLoaded to include relationships in the response only if they are loaded. This ensures that relationships are not fetched unnecessarily, improving performance.

    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'content' => $this->content,
    
            // Include relationship data only when loaded
            'author' => $this->whenLoaded('author', function () {
                return [
                    'id' => $this->author->id,
                    'name' => $this->author->name,
                ];
            }),
    
            // Include nested relationships
            'comments' => $this->whenLoaded('comments', function () {
                return $this->comments->map(function ($comment) {
                    return [
                        'id' => $comment->id,
                        'content' => $comment->content,
                        'user' => $comment->whenLoaded('user', function () use ($comment) {
                            return [
                                'id' => $comment->user->id,
                                'name' => $comment->user->name,
                            ];
                        }),
                    ];
                });
            }),
        ];
    }

Ensure relationships are loaded conditionally in the query:

    use App\Http\Resources\PostResource;
    use App\Models\Post;
    
    public function show($id)
    {
        $post = Post::with(['author', 'comments.user'])->findOrFail($id);
    
        return new PostResource($post);
    }

By default, Laravel wraps the resource response in a data key. You can disable or customize this behavior in the AppServiceProvider:

    use Illuminate\Http\Resources\Json\JsonResource;
    
    public function boot()
    {
        JsonResource::withoutWrapping();
    }

# A Note on HTTP Status Codes and the Response Format

- 200: OK. The standard success code and default option.
+ 201: Object created. Useful for the store actions.
* 204: No content. When an action was executed successfully, but there is no content to return.
- 206: Partial content. Useful when you have to return a paginated list of resources.
+ 400: Bad request. The standard option for requests that fail to pass validation.
+ 401: Unauthorized. The user needs to be authenticated.
+ 403: Forbidden. The user is authenticated, but does not have the permissions to perform an action.
+ 404: Not found. This will be returned automatically by Laravel when the resource is not found.
+ 500: Internal server error. Ideally you're not going to be explicitly returning this, but if something unexpected breaks, this is what your user is going to receive.
+ 503: Service unavailable. Pretty self explanatory, but also another code that is not going to be returned explicitly by the application.

# API Authentication and Authorization

## Authentication with Sanctum

**token authentication : **

        // routes/api.php
        use Illuminate\Support\Facades\Route;
        use App\Http\Controllers\AuthController;
        
        Route::post('/login', [AuthController::class, 'login']);
        Route::middleware('auth:sanctum')->get('/user', function (Request $request) {
            return $request->user();
        });

 **AuthController : **

        // app/Http/Controllers/AuthController.php
        namespace App\Http\Controllers;
        
        use Illuminate\Http\Request;
        use Illuminate\Support\Facades\Auth;
        
        class AuthController extends Controller
        {
            public function login(Request $request)
            {
                $credentials = $request->only('email', 'password');
                if (Auth::attempt($credentials)) {
                    return response()->json([
                        'token' => $request->user()->createToken('API Token')->plainTextToken,
                        'user' => Auth::user(),
                    ]);
                }
                return response()->json(['message' => 'Invalid credentials'], 401);
            }
        }


## Authorization with Policies

Laravel Policies allow fine-grained control over resource access.

**Policy for Post**

        php artisan make:policy PostPolicy --model=Post

**PostPolicy.php**

        // app/Policies/PostPolicy.php
        namespace App\Policies;
        
        use App\Models\Post;
        use App\Models\User;
        
        class PostPolicy
        {
            public function update(User $user, Post $post)
            {
                return $user->id === $post->user_id;
            }
        }

**Register Policy**

        // Register Policy in AuthServiceProvider
        protected $policies = [
            Post::class => PostPolicy::class,
        ];

**Usage**

        // Usage in Controller
        public function update(Request $request, Post $post)
        {
            $this->authorize('update', $post);
        
            $post->update($request->all());
            return response()->json(['message' => 'Post updated!']);
        }


# Rate Limiting and Throttling
## Laravel allows API rate limits using the RateLimiter facade.

**Custom Rate Limiting**

        // app/Providers/RouteServiceProvider.php
        use Illuminate\Support\Facades\RateLimiter;
        
        protected function configureRateLimiting()
        {
            RateLimiter::for('api', function (Request $request) {
                return $request->user()
                    ? Limit::perMinute(100)->by($request->user()->id)
                    : Limit::perMinute(10)->by($request->ip());
            });
        }

# Middleware

**Cusotm Middleware**

        php artisan make:middleware CheckApiKey

**Check Api key**

        // app/Http/Middleware/CheckApiKey.php
        public function handle($request, Closure $next)
        {
            if ($request->header('API_KEY') !== env('API_KEY')) {
                return response()->json(['message' => 'Invalid API Key'], 401);
            }
            return $next($request);
        }

**Rate Limiting by User Role**

        use Illuminate\Http\Request;
        
        public function configureRateLimiting()
        {
            RateLimiter::for('api', function (Request $request) {
                return $request->user()->is_admin
                    ? Limit::perMinute(100)
                    : Limit::perMinute(20);
            });
        }

# API Versioning

**Route Prefix**

        Route::prefix('v1')->group(function () {
            Route::get('/posts', [PostController::class, 'index']);
        });
        
        Route::prefix('v2')->group(function () {
            Route::get('/posts', [PostController::class, 'newIndex']);
        });

**Versioning with Headers**

        Route::middleware('checkVersion:v1')->group(function () {
        Route::get('/posts', [PostController::class, 'index']);
        });

# Caching for Performance

Tagged caching is helpful when you need to group related cached items for easier management.

        use Illuminate\Support\Facades\Cache;
        
        public function getPosts()
        {
            $posts = Cache::tags(['posts', 'api'])->remember('all_posts', 60, function () {
                return Post::all();
            });
        
            return response()->json($posts);
        }
        
        // Clear cache for specific tags
        Cache::tags(['posts'])->flush();
        

# Scheduled Tasks with APIs

**Dynamic API Scheduling**
Call external APIs periodically and update database.


        // app/Console/Kernel.php
        protected function schedule(Schedule $schedule)
        {
            $schedule->call(function () {
                $response = Http::get('https://api.example.com/updates');
                $data = $response->json();
        
                // Save data to database
                foreach ($data as $item) {
                    Post::updateOrCreate(['id' => $item['id']], $item);
                }
            })->dailyAt('12:00');
        }

#  API Localization

**Define Translations**

        // resources/lang/en/messages.php
        return [
            'welcome' => 'Welcome!',
        ];
        
        // resources/lang/bn/messages.php
        return [
            'welcome' => 'স্বাগতম',
        ];

**Use Translations in API**

        use Illuminate\Support\Facades\App;
        
        public function index()
        {
            App::setLocale('bn'); // Dynamically set locale
            return response()->json(['message' => __('messages.welcome')]);
        }

# API security

**Validating Request Signatures**
Ensure requests come from trusted sources by validating signatures.

        public function handle($request, Closure $next)
        {
            $signature = $request->header('X-Signature');
            $computedSignature = hash_hmac('sha256', $request->getContent(), env('API_SECRET'));
        
            if (!hash_equals($computedSignature, $signature)) {
                return response()->json(['message' => 'Invalid signature'], 401);
            }
        
            return $next($request);
        }
**CORS Configuration**
Control which domains can access your API by setting up CORS in config/cors.php.

        'paths' => ['api/*'],
        'allowed_methods' => ['*'],
        'allowed_origins' => ['https://example.com'],
        'allowed_headers' => ['*'],
        'exposed_headers' => [],
        'max_age' => 0,
        'supports_credentials' => false,

**Protect Against SQL Injection**
Use Eloquent or Query Builder to prevent SQL injection. Avoid raw SQL queries unless sanitized properly.

        // Secure query
        $users = User::where('email', $request->email)->get();
        
        // Vulnerable example (avoid)
        DB::select("SELECT * FROM users WHERE email = '{$request->email}'");

**Audit Logging**
Log API requests and responses to monitor suspicious activities.

        public function handle($request, Closure $next)
        {
            Log::info('API Request:', $request->all());
            $response = $next($request);
            Log::info('API Response:', $response->getContent());
            return $response;
        }

**Content Security Policy (CSP)**
Implement CSP headers to prevent XSS attacks:

        header('Content-Security-Policy: default-src \'self\'');

**Restrict API Routes by IP**

        // app/Http/Middleware/CheckIP.php
        public function handle($request, Closure $next)
        {
            $allowedIps = ['127.0.0.1', '192.168.1.1'];
            if (!in_array($request->ip(), $allowedIps)) {
                return response()->json(['message' => 'Unauthorized'], 403);
            }
            return $next($request);
        }
