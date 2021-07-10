# Apache
## Apache FrontEnd config
```
    RewriteEngine On
    RewriteCond %{REQUEST_FILENAME}.php -f`
    RewriteRule (.*) $1.php [L]
    RewriteCond %{REQUEST_FILENAME}.html -f
    RewriteRule (.*) $1.html [L]
```
## Apache BackEnd config RESTful API-hoz
```
RewriteEngine On
RewriteBase /api/

RewriteRule ^/index\.php$ - [L,NC]

RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . index.php [L]
```
## VirtualHost config
```
NameVirtualHost *
    <VirtualHost *>
        DocumentRoot "C:/xampp/htdocs"
        ServerName localhost
    </VirtualHost>
    <VirtualHost *>
        ServerName www.shanghai.com
        ServerAlias shanghai.com
        DocumentRoot "C:/xampp/htdocs/php-api"
    </VirtualHost>
```
## httpd.conf
```
<Directory "C:/xampp/htdocs/php-api">
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
```

# Backend
## index.php
```php
<?php
require_once 'app.php';

header("Access-Control-Allow-Origin: *");
header("Content-Type: application/json; charset=UTF-8");
header("Access-Control-Allow-Methods: GET,POST,PUT,DELETE");

$uri = parse_url(substr($_SERVER['REQUEST_URI'], 1), PHP_URL_PATH);
$uri = explode('/', $uri);

if ( $uri[0] !== "api" )
{
    header("HTTP/1.1 404 Not Found");
    exit();
}

$db = [
    "driver"    => "mysql",
    "host"      => "127.0.0.1",
    "database"  => "db",
    "charset"   => "utf8",
    "username"  => "root",
    "password"  => ""
];

$request = [
    "method"    => $_SERVER['REQUEST_METHOD'],
    "url"       => array_slice( $uri, 1 ),
    "params"    => $_REQUEST
];


$app = new Router( $request );
$app->dbConnect( $db );

require_once 'routes.php';

$app->run();
```
## routes.php
```php
<?php

$app->register( "GET", "/buildings", function( $request, $database ) {
    $query = $database->prepare("SELECT * FROM departments");
    $query->execute();

    return $query->fetchAll();
});
```
## app.php
```php
<?php

class UrlFactory
{
    private $url = [];

    public function __construct( $uri )
    {
        $parsed = explode( '/', $uri );
        foreach( $parsed as $item )
        {
            if ( strlen( $item ) > 1 )
            {
                $this->url[$item] = (strpos( $item, '{' ) !== false && strpos( $item, '}' ) !== false) ? 0 : 1;
            }
        }
    }

    public function equals( $urlArray )
    {
        if ( count($this->url) !== count($urlArray) ) return false;

        $i = 0;
        foreach( $this->url as $key => $value )
        {
            // if ( $value == 0 )
            // if ( $value == 1 && $key == $urlArray[$i] )
            if ( $value == 1 && $key !== $urlArray[$i] ) return false;
            

            $i++;
        }

        return true;
    }
}

class Router
{
    private $cluster = [];
    private $request;
    private $database;


    public function __construct( $request )
    {
        $this->request = $request;
    }

    public function dbConnect( $config )
    {
        $this->database = new PDO(
            "{$config['driver']}:dbname={$config['database']};host={$config['host']};charset={$config['charset']}",
            $config['username'],
            $config['password']
        );
    }

    public function register( string $method, string $path, callable $callback )
    {
        array_push( $this->cluster, [
            "url" => new UrlFactory( $path ),
            "method" => $method,
            "callback" => $callback
        ]);
    }

    public function run()
    {
        foreach( $this->cluster as $current )
        {
            if ( $current["url"]->equals( $this->request["url"] ) )
            {
                if ( $current["method"] !== $this->request["method"] )
                {
                    header("HTTP/1.1 405 Method Not Allowed");
                    echo json_encode( "405 Method Not Allowed" );
                    exit();
                }

                $response = call_user_func_array( $current["callback"], array( $this->request, $this->database ) );
                echo json_encode( $response );
                exit();
            }
        }

        header("HTTP/1.1 404 Not Found");
        echo json_encode( "404 Not Found" );
        exit();
    }
}
```
