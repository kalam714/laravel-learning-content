# Indexing 

**What is Indexing in a Database?**

Indexing is a technique used in databases to speed up the retrieval of data from tables. It works like an index in a book—helping the database find rows quickly without scanning the entire table.

**How Indexing Works**

When a database query searches for data, it usually scans every row in the table (Full Table Scan), which is slow for large datasets.
An index creates a sorted data structure (e.g., B-Tree, Hash) that allows the database to find rows much faster.

👉 Think of it like this:

+ Without an index: Searching for a word in a 1000-page book by reading every page.
+ With an index: Using the book's index to jump directly to the relevant pages.

**Types of Indexes**

**1.Primary Index (Clustered Index)**

+ Created automatically on the primary key.
+ The table rows are physically stored based on this index.

      CREATE TABLE users (
          id INT PRIMARY KEY,
          name VARCHAR(100),
          email VARCHAR(100) UNIQUE
      );
+ Here, id is the clustered index (default for primary keys).

**2.Unique Index**

+ Ensures no duplicate values in a column.

      CREATE UNIQUE INDEX idx_users_email ON users(email);

**3.Composite Index (Multi-Column Index)**

+ Index on multiple columns for queries using both fields.


      CREATE INDEX idx_users_name_email ON users(name, email);
      
      // Useful for queries like:
      
      SELECT * FROM users WHERE name = 'John' AND email = 'john@example.com';

**4.Full-Text Index**

+ Used for searching text in large datasets.

      CREATE FULLTEXT INDEX idx_posts_content ON posts(content);


**Use Indexing?**

✅ When searching large datasets frequently.
✅ When filtering results using WHERE clauses.
✅ When sorting (ORDER BY) or joining large tables.
❌ Avoid indexing small tables (overhead isn't worth it).
❌ Avoid excessive indexes (too many can slow down updates).

**Add an Index in Migration**

      use Illuminate\Database\Migrations\Migration;
      use Illuminate\Database\Schema\Blueprint;
      use Illuminate\Support\Facades\Schema;
      
      class CreateNewsTable extends Migration
      {
          public function up()
          {
              Schema::create('news', function (Blueprint $table) {
                  $table->id();
                  $table->string('title');
                  $table->text('content');
                  $table->date('published_at')->index(); // Adding an index here
                  $table->timestamps();
              });
          }
      
          public function down()
          {
              Schema::dropIfExists('news');
          }
      }


# Partitioning

**What is Partitioning in a Database?**

Partitioning is a technique used in databases to divide a large table into smaller, more manageable pieces (partitions) while keeping them as part of a single logical table. This helps improve performance, query speed, and manageability.

**Why Use Partitioning?**

+ Improves query performance by scanning only relevant partitions.
+ Speeds up data retrieval for large datasets.
+ Makes maintenance easier (e.g., archiving old data).
+ Improves load balancing and scalability in distributed databases.

**Types of Partitioning**


**1.Range Partitioning**

+ Data is divided based on a range of values in a column.
+ Example: Partitioning a sales table by year.

      CREATE TABLE sales (
          id INT,
          order_date DATE,
          amount DECIMAL(10,2)
      ) PARTITION BY RANGE (YEAR(order_date)) (
          PARTITION p1 VALUES LESS THAN (2022),
          PARTITION p2 VALUES LESS THAN (2023),
          PARTITION p3 VALUES LESS THAN (2024)
      );

+ Queries searching for order_date = '2023-06-15' will only scan partition p2.

**2.List Partitioning**

+ Data is divided based on a list of predefined values.
+ Example: Partitioning an employees table by department.

      CREATE TABLE employees (
          id INT,
          name VARCHAR(100),
          department VARCHAR(50)
      ) PARTITION BY LIST (department) (
          PARTITION p_hr VALUES IN ('HR'),
          PARTITION p_sales VALUES IN ('Sales'),
          PARTITION p_it VALUES IN ('IT')
      );

+ A query filtering WHERE department = 'HR' will scan only the HR partition.

**3.Hash Partitioning**

+ Data is divided using a hash function, ensuring even distribution.
+ Useful for load balancing.

      CREATE TABLE users (
          id INT NOT NULL,
          name VARCHAR(100)
      ) PARTITION BY HASH(id) PARTITIONS 4;
  
+ The system automatically distributes data across 4 partitions.
  

**4.Key Partitioning**

+ Similar to Hash Partitioning but uses a primary key

      CREATE TABLE orders (
          order_id INT PRIMARY KEY,
          customer_id INT
      ) PARTITION BY KEY(order_id) PARTITIONS 3;

+ MySQL automatically assigns rows based on the order_id key.
  

**5.Composite Partitioning**

+ Combination of Multiple Methods
+ Example: Range + Hash Partitioning

      CREATE TABLE logs (
          id INT,
          log_date DATE,
          server_id INT
      ) PARTITION BY RANGE (YEAR(log_date))
      SUBPARTITION BY HASH(server_id) SUBPARTITIONS 4 (
          PARTITION p1 VALUES LESS THAN (2022),
          PARTITION p2 VALUES LESS THAN (2023)
      );

+ First, data is partitioned by year.
+ Then, each partition is further divided into 4 subpartitions using server_id.

**When Should You Use Partitioning?**

+ When dealing with large datasets (millions of rows).
+ When queries frequently filter based on specific columns (e.g., date, category).
+ When archiving old data without affecting performance.
+ When needing even data distribution in large-scale applications.

# Using Partitioning in Laravel 

**Define Partitioned Table in Migration**

    use Illuminate\Database\Migrations\Migration;
    use Illuminate\Support\Facades\DB;
    
    class CreateNewsTable extends Migration
    {
        public function up()
        {
            DB::statement("
                CREATE TABLE news (
                    id INT PRIMARY KEY AUTO_INCREMENT,
                    title VARCHAR(255) NOT NULL,
                    content TEXT NOT NULL,
                    published_at DATE NOT NULL
                ) PARTITION BY RANGE (YEAR(published_at)) (
                    PARTITION p_2022 VALUES LESS THAN (2023),
                    PARTITION p_2023 VALUES LESS THAN (2024),
                    PARTITION p_2024 VALUES LESS THAN (2025),
                    PARTITION p_future VALUES LESS THAN MAXVALUE
                )
            ");
        }
    
        public function down()
        {
            DB::statement("DROP TABLE IF EXISTS news");
        }
    }


**Query Data from Partitions**

    $news2023 = News::whereYear('published_at', 2023)->get();
    
    //Or, if you want to directly access partitions, use raw queries:
    
    $year = 2023;
    $news = DB::select("SELECT * FROM news PARTITION (p_$year)");




