To utilize this class, first import Mysqldbi.php into your project, and require it.

```php
require_once('Mysqlidb.php');
```

After that, create a new instance of the class.

```php
$db = new Mysqlidb('host', 'username', 'password', 'databaseName');
```

Its also possible to set a table prefix:
```php
$db->setPrefix('tablePrefix');
```

Next, prepare your data, and call the necessary methods. 

### Insert Query
Simple example
```php
$data = Array ("login" => "admin",
               "firstName" => "John",
               "lastName" => 'Doe'
)
$id = $db->insert('users', $data);
if($id)
    echo 'user was created. Id='.$id;
```

Insert with functions use
```php
$data = Array(
	'login' => 'admin',
    'active' => true,
	'firstName' => 'John',
	'lastName' => 'Doe',
	'password' => $db->func('SHA1(?)',Array ("secretpassword+salt")),
	// password = SHA1('secretpassword+salt')
	'createdAt' => $db->now(),
	// createdAt = NOW()
	'expires' => $db->now('+1Y')
	// expires = NOW() + interval 1 year
	// Supported intervals [s]econd, [m]inute, [h]hour, [d]day, [M]onth, [Y]ear
);

$id = $db->insert('users', $data);
if($id)
    echo 'user was created. Id='.$id;
```

### Update Query
```php
$data = Array (
	'firstName' => 'Bobby',
	'lastName' => 'Tables',
	'editCount' => $db->inc(2),
	// editCount = editCount + 2;
	'active' => $db->not()
	// active = !active;
);
$db->where('id', 1);
if($db->update('users', $data)) echo 'successfully updated'; 
```

### Select Query
After any select/get function calls amount or returned rows
is stored in $count variable
```php
$users = $db->get('users'); //contains an Array of all users 
$users = $db->get('users', 10); //contains an Array 10 users
```

or select with custom columns set. Functions also could be used

```php
$stats = $db->getOne ("users", "sum(id), count(*) as cnt");
echo "total ".$stats['cnt']. "users found";

$cols = Array ("id, name, email");
$users = $db->get ("users", null, $cols);
if ($db->count > 0)
    foreach ($users as $user) { 
        print_r ($user);
    }
```

or select just one row

```php
$db->where ("id", 1);
$user = $db->getOne ("users");
echo $user['id'];
```

### Delete Query
```php
$db->where('id', 1);
if($db->delete('users')) echo 'successfully deleted';
```

### Generic Query Method
```php
$users = $db->rawQuery('SELECT * from users');
foreach ($users as $user) {
    print_r ($user);
}
```

### Raw Query Method
```php
$params = Array(1, 'admin');
$users = $db->rawQuery("SELECT id, firstName, lastName FROM users WHERE id = ? AND login = ?", $params);
print_r($users); // contains Array of returned rows

// will handle any SQL query
$params = Array(10, 1, 10, 11, 2, 10);
$resutls = $db->rawQuery("(SELECT a FROM t1 WHERE a = ? AND B = ? ORDER BY a LIMIT ?) UNION(SELECT a FROM t2 WHERE a = ? AND B = ? ORDER BY a LIMIT ?)", $params);
print_r($results); // contains Array of returned rows
```


### Where Method
This method allows you to specify the parameters of the query.

Regular == operator:
```php
$db->where('id', 1);
$db->where('login', 'admin');
$results = $db->get('users');
// Gives: SELECT * FROM users WHERE id=1 AND login='admin';
```

```php
$db->where('id', 50, ">=");
// or $db->where('id', Array('>=' => 50));

$results = $db->get('users');
// Gives: SELECT * FROM users WHERE id >= 50;
```

BETWEEN / NOT BETWEEN:
```php
$db->where('id', Array(4, 20), 'between');
// or $db->where('id', Array('between' => Array(4, 20) ) );

$results = $db->get('users');
// Gives: SELECT * FROM users WHERE id BETWEEN 4 AND 20
```

IN / NOT IN:
```php
$db->where('id', Array(1, 5, 27, -1, 'd'), 'IN');
// or $db->where('id', Array( 'in' => Array(1, 5, 27, -1, 'd') ) );

$results = $db->get('users');
// Gives: SELECT * FROM users WHERE id IN (1, 5, 27, -1, 'd');
```

OR CASE
```php
$db->where ('firstName', 'John');
$db->orWhere ('firstName', 'Peter');
$results = $db->get ('users');
// Gives: SELECT * FROM users WHERE firstName='John' OR firstName='peter'
```

NULL comparison:
```php
$db->where ("lastName", NULL, '<=>');
$results = $db->get("users");
// Gives: SELECT * FROM users where lastName <=> NULL
```

Also you can use raw where conditions:
```php
$db->where ("id != companyId");
$results = $db->get("users");
```

Or raw condition with variables:
```php
$db->where("id = ? or id = ?", Array(6,2));
$res = $db->get ("users");
// Gives: SELECT * FROM users WERE id = 2 or id = 2;
```


Optionally you can use method chaining to call where multiple times without referencing your object over an over:

```php
$results = $db
	->where('id', 1)
	->where('login', 'admin')
	->get('users');
```

### Ordering method
```php
$db->orderBy("id","asc");
$db->orderBy("login","Desc");
$results = $db->get('users');
// Gives: SELECT * FROM users ORDER BY id ASC,login DESC;
```

### Grouping method
```php
$db->groupBy ("name");
$results = $db->get ('users');
// Gives: SELECT * FROM users GROUP BY name;
```

Join table products with table users with LEFT JOIN by tenantID
### JOIN method
```php
$db->join("users u", "p.tenantID=u.tenantID", "LEFT");
$db->where("u.id", 6);
$products = $db->get ("products p", null, "u.name, p.productName");
print_r ($products);
```

### Properties sharing
Its is also possible to copy properties
```php
$db->where ("agentId", 10);

$customers = $common->copy ();
$res = $customers->get ("customers");
// SELECT * FROM customers where agentId = 10

$db->orWhere ("agentId", 20);
$res = $db->get ("users");
// SELECT * FROM users where agentId = 10 or agentId = 20
```

### Subqueries
Subquery in selects:
```php
$ids = $db->subQuery ();
$ids->where ("qty", 2, ">");
$ids->get ("products", null, "userId");

$db->where ("id", $ids, 'in');
$res = $db->get ("users");
// Gives SELECT * FROM users WHERE id IN (SELECT userId FROM products WHERE qty > 2)
```

Subquery in inserts:
```php
$userIdQ = $db->subQuery ();
$userIdQ->where ("id", 6);
$userIdQ->getOne ("users", "name"),

$data = Array (
    "productName" => "test product",
    "userId" => $userIdQ,
    "lastUpdated" => $db->now()
);
$id = $db->insert ("products", $data);
// Gives INSERT INTO PRODUCTS (productName, userId, lastUpdated) values ("test product", (SELECT name FROM users WHERE id = 6), NOW());
```
UNION queries
```php
$common = $db->subQuery();
$common->where ("agentId", 10);

$customers = $common->copy();
$customers->get ("customers");

$users = $common->copy();
$users->get ("users");

$db->orderBy ("lastModified");
$res = $db->union (Array ($customers, $users, $companies), "ALL");
GIVES (SELECT * FROM customers WHERE agentId = 10 ) UNION ALL (SELECT * FROM users WHERE agentId = 10 ) ORDER BY lastModified DESC
```
### Helper commands
Reconnect in case mysql connection died
```php
if (!$db->ping())
    $db->connect()
```

Obtain an initialized instance of the class from another class
```php
    $db = MysqliDb::getInstance();
```

Get last executed SQL query.
Please note that function returns SQL query only for debugging purposes as its execution most likely will fail due missing quotes around char variables.
```php
    $db->get('users');
    echo "Last executed query was ". $db->getLastQuery();
```
