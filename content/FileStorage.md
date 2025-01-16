# Laravel File Storage

Laravel's file storage system provides a simple and unified API to interact with different storage services such as local files, Amazon S3, FTP, and more. The system is built on top of the Flysystem PHP package, which offers drivers for various file systems

**File System Configuration**: Laravel stores all file system configuration in the config/filesystems.php file. This file contains default storage settings, as well as settings for each supported disk.

Example of filesystems.php:


    return [
        'default' => env('FILESYSTEM_DISK', 'local'),
    
        'disks' => [
            'local' => [
                'driver' => 'local',
                'root' => storage_path('app'),
            ],
    
            'public' => [
                'driver' => 'local',
                'root' => public_path('storage'),
                'url' => env('APP_URL').'/storage',
                'visibility' => 'public',
            ],
    
            's3' => [
                'driver' => 's3',
                'key' => env('AWS_ACCESS_KEY_ID'),
                'secret' => env('AWS_SECRET_ACCESS_KEY'),
                'region' => env('AWS_DEFAULT_REGION'),
                'bucket' => env('AWS_BUCKET'),
                'url' => env('AWS_URL'),
            ],
        ],
    ];

You can modify this configuration to specify your default disk, as well as settings for multiple disks.

**The Local Driver**

The local disk is the default driver in Laravel, which stores files on the server’s local file system. You can interact with files using Laravel's Storage facade.

    use Illuminate\Support\Facades\Storage;
    
    Storage::disk('local')->put('file.txt', 'Contents of the file');

**The Public Disk**

The public disk is used for storing files that should be publicly accessible. It is typically configured to store files in the public/storage directory, which should be linked to the storage/app/public directory via the php artisan storage:link command.

    Storage::disk('public')->put('file.txt', 'Contents of the file');

**Driver Prerequisites**

Each storage driver has specific prerequisites:

+ Local Driver: No extra configuration is required.
+ Amazon S3: You must have an AWS account, access keys, and a valid bucket.
+ FTP: Requires FTP server credentials.

**Scoped and Read-Only Filesystems**

Scoped filesystems allow you to limit the scope of files. For instance, you can limit access to a specific folder on a disk. You can also configure read-only disks, where files cannot be deleted or written to.

    'disks' => [
        'read_only' => [
            'driver' => 'local',
            'root' => storage_path('app/public'),
            'read_only' => true,
        ],
    ],

**Amazon S3 Compatible Filesystems**

Laravel supports not only Amazon S3 but also other S3-compatible services, such as MinIO, DigitalOcean Spaces, and more. You just need to provide the correct configuration details in filesystems.php.

    'disks' => [
        's3' => [
            'driver' => 's3',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'region' => env('AWS_DEFAULT_REGION'),
            'bucket' => env('AWS_BUCKET'),
            'url' => env('AWS_URL'),
        ],
    ],

**Obtaining Disk Instances**

You can use the Storage facade to obtain instances of different disks:

    $disk = Storage::disk('s3');  // For S3 disk
    $disk = Storage::disk('local');  // For Local disk

**On-Demand Disks**

You can dynamically configure disks at runtime using config():
    
    $disk = Storage::disk(config('filesystems.default'));

**Retrieving,download and url generate**


    // To retrieve files, use the get method:
    
    $content = Storage::disk('local')->get('file.txt');

    // To download files, you can use the download method:
    
    return Storage::download('file.txt');

    // You can generate URLs for files stored in publicly accessible disks:

    $url = Storage::disk('public')->url('file.txt');



**Temporary URLs**

Temporary URLs provide a limited-time access link to a file. Useful for private files stored in S3:

    $url = Storage::disk('s3')->temporaryUrl('file.txt', now()->addMinutes(5));

**File Metadata**

You can retrieve file metadata such as size, MIME type, and modification time:

    $size = Storage::disk('local')->size('file.txt');
    $type = Storage::disk('local')->mimeType('file.txt');
    $time = Storage::disk('local')->lastModified('file.txt');

**Storing Files**


    // You can store files from an uploaded request:

    $request->file('avatar')->store('avatars');

    // To specify a disk and directory:

    $request->file('avatar')->storeAs('avatars', 'avatar.jpg', 'public');

**Prepending and Appending To Files**

Laravel allows appending or prepending content to an existing file:

    Storage::disk('local')->append('file.txt', 'New content');
    Storage::disk('local')->prepend('file.txt', 'Start of file');

**Copying and Moving Files**

You can copy or move files between disks or directories:

    Storage::disk('local')->copy('file.txt', 'file_copy.txt');
    Storage::disk('local')->move('file.txt', 'file_moved.txt');

**Automatic Streaming**

For large files, Laravel supports automatic streaming, which avoids loading the entire file into memory. For instance, streaming a file to the browser:

    return Storage::download('largefile.txt');

**File Uploads**

To handle file uploads, use Laravel's file() or file method:
    
    $file = $request->file('image');
    $file->store('images');

**File Visibility**

Files stored in Laravel can have visibility attributes (public or private). This is important when you’re dealing with public files on the disk:

    Storage::disk('public')->setVisibility('file.txt', 'public');
    
**Deleting Files**

    Storage::disk('local')->delete('file.txt');
    
**Directories**

You can check if a directory exists or create it:

    Storage::makeDirectory('directory_name');
    Storage::deleteDirectory('directory_name');
    
**Testing**

Laravel provides the Storage::fake() method to simulate file uploads during testing. It prevents files from actually being stored.

    Storage::fake('local');
    
    $file = UploadedFile::fake()->image('avatar.jpg');
    
    $file->store('avatars');
    
    Storage::disk('local')->assertExists('avatars/avatar.jpg');
    
**Custom Filesystems**

You can add custom filesystems by extending the FilesystemAdapter and configuring them in the filesystems.php config.

Example custom filesystem:

    use League\Flysystem\Filesystem;
    use League\Flysystem\Adapter\Local;
    
    $adapter = new Local(storage_path('custom'));
    $filesystem = new Filesystem($adapter);

After creating a custom adapter, you can add it to the disks configuration.

    'disks' => [
        'custom' => [
            'driver' => 'custom',
            'adapter' => new CustomAdapter(),
        ],
    ],

