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

# API Errors and Exceptions
## Handling errors and exceptions effectively in Laravel APIs ensures that users receive clear, standardized, and informative responses. Laravel provides various tools and methods for handling API errors and exceptions

Laravel includes an App\Exceptions\Handler class where you can customize exception handling.
Example: Customizing Exception Responses
Edit the render method in Handler.php to handle API exceptions:
