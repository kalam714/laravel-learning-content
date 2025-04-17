# Multi-Tenant Laravel

**What is Multi-Tenancy?**

Multi-tenancy is a software architecture where a single instance of an application serves multiple customers (tenants). Each tenant's data is isolated and remains invisible to other tenants.
Think of it like an apartment building:

+ The building is your application
+ Each apartment is a tenant (a customer/organization)
+ Tenants share the building infrastructure but have private spaces

Common examples of multi-tenant applications include:

+ Slack (each workspace is a tenant)
+ Shopify (each store is a tenant)
+ Office 365 (each company is a tenant)

**Types of Multi-Tenancy in Laravel**

There are three main approaches to implementing multi-tenancy:

+ Database Separation: Each tenant gets their own database
+ Schema Separation: Each tenant gets their own schema within a shared database
+ Row-Level Separation: All tenants share the same database and tables, but rows are filtered by a tenant identifier

# Approach 1: Row-Level Separation (Single Database)

**Step 1: Create a Tenant Model**

    php artisan make:model Tenant -m

This creates a Tenant model and a migration file. Let's modify the migration:

    // database/migrations/xxxx_xx_xx_create_tenants_table.php
    public function up()
    {
        Schema::create('tenants', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('domain')->unique();
            $table->timestamps();
        });
    }

**Step 2: Create a User Model with Tenant Relationship**

Modify your user migration to include a tenant_id:

    // database/migrations/xxxx_xx_xx_create_users_table.php
    public function up()
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->foreignId('tenant_id')->constrained();
            $table->rememberToken();
            $table->timestamps();
        });
    }

Update your User model:

    // app/Models/User.php
    namespace App\Models;
    
    use Illuminate\Foundation\Auth\User as Authenticatable;
    use Illuminate\Notifications\Notifiable;
    
    class User extends Authenticatable
    {
        use Notifiable;
    
        protected $fillable = [
            'name', 'email', 'password', 'tenant_id',
        ];
    
        public function tenant()
        {
            return $this->belongsTo(Tenant::class);
        }
    }

**Step 3: Create a Tenant Scope**

Create a global scope that automatically filters queries by tenant_id:

    php artisan make:scope TenantScope

Then modify it:

    // app/Models/Scopes/TenantScope.php
    namespace App\Models\Scopes;
    
    use Illuminate\Database\Eloquent\Builder;
    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Database\Eloquent\Scope;
    
    class TenantScope implements Scope
    {
        public function apply(Builder $builder, Model $model)
        {
            if (session()->has('tenant_id')) {
                $builder->where('tenant_id', session('tenant_id'));
            }
        }
    }

**Step 4: Create a Trait for Tenant Models**

    // app/Models/Traits/BelongsToTenant.php
    namespace App\Models\Traits;
    
    use App\Models\Scopes\TenantScope;
    use App\Models\Tenant;
    
    trait BelongsToTenant
    {
        protected static function bootBelongsToTenant()
        {
            static::addGlobalScope(new TenantScope);
            
            static::creating(function ($model) {
                if (session()->has('tenant_id')) {
                    $model->tenant_id = session('tenant_id');
                }
            });
        }
    
        public function tenant()
        {
            return $this->belongsTo(Tenant::class);
        }
    }

**Step 5: Apply the Trait to Models**

Now apply the trait to any model that should be tenant-aware:

    // app/Models/Post.php (example)
    namespace App\Models;
    
    use App\Models\Traits\BelongsToTenant;
    use Illuminate\Database\Eloquent\Model;
    
    class Post extends Model
    {
        use BelongsToTenant;
        
        protected $fillable = ['title', 'content'];
    }

**Step 6: Create a Middleware to Set Current Tenant**

    php artisan make:middleware SetTenantMiddleware


    // app/Http/Middleware/SetTenantMiddleware.php
    namespace App\Http\Middleware;
    
    use App\Models\Tenant;
    use Closure;
    use Illuminate\Http\Request;
    
    class SetTenantMiddleware
    {
        public function handle(Request $request, Closure $next)
        {
            // Get domain from request
            $host = $request->getHost();
            
            // Find tenant by domain
            $tenant = Tenant::where('domain', $host)->first();
            
            if ($tenant) {
                // Set the tenant ID in the session
                session(['tenant_id' => $tenant->id]);
            }
            
            return $next($request);
        }
    }


**Step 7: Testing Row-Level Multi-Tenancy**

    // Create tenants
    $tenant1 = Tenant::create(['name' => 'Acme Inc', 'domain' => 'acme.localhost']);
    $tenant2 = Tenant::create(['name' => 'Globex Corp', 'domain' => 'globex.localhost']);
    
    // Create users for each tenant
    $user1 = User::create([
        'name' => 'John Doe',
        'email' => 'john@acme.localhost',
        'password' => bcrypt('password'),
        'tenant_id' => $tenant1->id
    ]);
    
    $user2 = User::create([
        'name' => 'Jane Smith',
        'email' => 'jane@globex.localhost',
        'password' => bcrypt('password'),
        'tenant_id' => $tenant2->id
    ]);
    
    // Switch to tenant1 context
    session(['tenant_id' => $tenant1->id]);
    $users = User::all(); // Should only include users from tenant1
    
    // Switch to tenant2 context
    session(['tenant_id' => $tenant2->id]);
    $users = User::all(); // Should only include users from tenant2

# Approach 2: Separate Databases per Tenant

For more robust isolation, each tenant can have its own database.

**Step 1: Create a Config for Databases**

    // config/tenancy.php
    return [
        'database' => [
            'prefix' => 'tenant_',
            'suffix' => '',
        ],
    ];

**Step 2: Modify DatabaseManager to Switch Connections**

This approach requires more advanced setup. Let's create a service to manage tenant database connections:

    php artisan make:provider TenancyServiceProvider


    // app/Providers/TenancyServiceProvider.php
    namespace App\Providers;
    
    use App\Models\Tenant;
    use Illuminate\Support\ServiceProvider;
    use Illuminate\Support\Facades\DB;
    use Illuminate\Support\Facades\Config;
    
    class TenancyServiceProvider extends ServiceProvider
    {
        public function register()
        {
            //
        }
    
        public function boot()
        {
            // Register the tenant connection resolver
            $this->app->singleton('tenant.connection', function () {
                return new TenantConnectionResolver();
            });
            
            // Set up tenant database switching when a tenant is identified
            $this->app['events']->listen('tenancy.tenant.identified', function (Tenant $tenant) {
                $this->switchToTenant($tenant);
            });
        }
        
        protected function switchToTenant(Tenant $tenant)
        {
            $prefix = config('tenancy.database.prefix');
            $databaseName = $prefix . $tenant->id;
            
            Config::set('database.connections.tenant', [
                'driver' => 'mysql',  // You can use any driver
                'host' => env('DB_HOST', '127.0.0.1'),
                'port' => env('DB_PORT', '3306'),
                'database' => $databaseName,
                'username' => env('DB_USERNAME', 'forge'),
                'password' => env('DB_PASSWORD', ''),
                'unix_socket' => env('DB_SOCKET', ''),
                'charset' => 'utf8mb4',
                'collation' => 'utf8mb4_unicode_ci',
                'prefix' => '',
                'prefix_indexes' => true,
                'strict' => true,
                'engine' => null,
                'options' => extension_loaded('pdo_mysql') ? array_filter([
                    \PDO::MYSQL_ATTR_SSL_CA => env('MYSQL_ATTR_SSL_CA'),
                ]) : [],
            ]);
            
            // Change the default connection to the tenant connection
            DB::purge('tenant');
            Config::set('database.default', 'tenant');
        }
    }

**Step 3: Create a Command to Create Tenant Databases**

    php artisan make:command CreateTenantDatabase

    // app/Console/Commands/CreateTenantDatabase.php
    namespace App\Console\Commands;
    
    use App\Models\Tenant;
    use Illuminate\Console\Command;
    use Illuminate\Support\Facades\DB;
    
    class CreateTenantDatabase extends Command
    {
        protected $signature = 'tenant:create-database {tenant}';
        protected $description = 'Create a database for a tenant';
    
        public function handle()
        {
            $tenantId = $this->argument('tenant');
            $tenant = Tenant::find($tenantId);
            
            if (!$tenant) {
                $this->error("Tenant with ID {$tenantId} not found.");
                return 1;
            }
            
            $prefix = config('tenancy.database.prefix');
            $databaseName = $prefix . $tenant->id;
            
            // Create the database
            DB::statement("CREATE DATABASE IF NOT EXISTS {$databaseName}");
            
            $this->info("Database {$databaseName} created successfully!");
            
            // Run migrations on the new database
            $this->call('tenants:migrate', [
                'tenant' => $tenant->id
            ]);
            
            return 0;
        }
    }


**Step 4: Create a Command to Run Migrations for Tenant Databases**

    php artisan make:command TenantMigrate

    // app/Console/Commands/TenantMigrate.php
    namespace App\Console\Commands;
    
    use App\Models\Tenant;
    use Illuminate\Console\Command;
    use Illuminate\Support\Facades\Artisan;
    
    class TenantMigrate extends Command
    {
        protected $signature = 'tenants:migrate {tenant?}';
        protected $description = 'Run migrations for tenant databases';
    
        public function handle()
        {
            $tenantId = $this->argument('tenant');
            
            if ($tenantId) {
                $this->migrateForTenant(Tenant::find($tenantId));
            } else {
                Tenant::all()->each(function ($tenant) {
                    $this->migrateForTenant($tenant);
                });
            }
            
            return 0;
        }
        
        protected function migrateForTenant(Tenant $tenant)
        {
            $prefix = config('tenancy.database.prefix');
            $database = $prefix . $tenant->id;
            
            $this->info("Migrating database for tenant {$tenant->name} (ID: {$tenant->id})");
            
            // Run migrations using the tenant connection
            $options = [
                '--database' => 'tenant',
                '--path' => 'database/migrations/tenant',
                '--force' => true,
            ];
            
            event('tenancy.tenant.identified', $tenant);
            Artisan::call('migrate', $options);
            
            $this->info("Migration complete for tenant {$tenant->name}");
        }
    }

# Advanced Concepts and Challenges


**1. Central vs. Tenant Databases**

Some data needs to be stored centrally (shared across tenants), while other data belongs to specific tenants:

    // Example central model
    class Plan extends Model
    {
        // Uses the default connection
        protected $connection = 'central';
    }
    
    // Example tenant model
    class Invoice extends Model
    {
        use BelongsToTenant; // or UsesTenantConnection with Spatie
    }


**2. Tenant Identification Strategies**

You can identify tenants in several ways:

+ Domain Based: tenant1.example.com, tenant2.example.com
+ Path Based: example.com/tenant1, example.com/tenant2
+ Request Parameter: example.com?tenant=tenant1
+ User Authentication: Identify tenant from the logged-in user

**3. Tenant Database Creation and Migration**

For separate databases approach, you need to manage:

+ Creating databases dynamically
+ Running migrations for each tenant database
+ Handling schema updates across all tenant databases

**4. Testing in a Multi-Tenant Environment**

Create test helpers to switch contexts:

    // tests/TestCase.php
    protected function actingAsTenant($tenant)
    {
        // For row-level approach
        session(['tenant_id' => $tenant->id]);
        
        // For separate database approach
        event('tenancy.tenant.identified', $tenant);
        
        return $this;
    }

**5. Queue Jobs and Multi-Tenancy**

When dispatching jobs, tenant context may be lost. Solutions:

    // Method 1: Add tenant ID to job constructor
    class ProcessInvoice implements ShouldQueue
    {
        protected $tenantId;
        
        public function __construct($tenantId)
        {
            $this->tenantId = $tenantId;
        }
        
        public function handle()
        {
            // Set tenant context
            session(['tenant_id' => $this->tenantId]);
            // or
            event('tenancy.tenant.identified', Tenant::find($this->tenantId));
            
            // Process the job...
        }
    }
    
    // Method 2: Use a middleware (with Spatie)
    ProcessInvoice::dispatch()->onConnection('redis')->withTenantContext();

























































