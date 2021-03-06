# About

[![Build Status](https://travis-ci.org/webinv/php-library-datatables.svg?branch=master)](https://travis-ci.org/webinv/php-library-datatables)
[![Latest Stable Version](https://poser.pugx.org/webinv/datatables/v/stable)](https://packagist.org/packages/webinv/datatables)
[![Total Downloads](https://poser.pugx.org/webinv/datatables/downloads)](https://packagist.org/packages/webinv/datatables)
[![Latest Unstable Version](https://poser.pugx.org/webinv/datatables/v/unstable)](https://packagist.org/packages/webinv/datatables)
[![License](https://poser.pugx.org/webinv/datatables/license)](https://packagist.org/packages/webinv/datatables)

Pure PHP models for datatables.net

* https://datatables.net
* https://datatables.net/manual/server-side

# Installation

`composer require webinv/datatables`

# Usage


## Basic array

```bash
cd examples/array
php -S localhost:8000

# Open browser http://localhost:8000/
```

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';

use Webinv\Datatables\RequestFactory;
use Webinv\Datatables\Response;

// Example data
$data = [
    [
        'first_name' => 'Tiger Nixon',
        'last_name' => 'System Architect',
        'position' => 'Edinburgh',
        'office' => '5421',
        'start_date' => '2011/04/25',
        'salary' => '$320,800'
    ],
    [
        'first_name' => 'Garrett Winters',
        'last_name' => 'Accountant',
        'position' => 'Tokyo',
        'office' => '8422',
        'start_date' => '2011/07/25',
        'salary' => '$170,750'
    ],
    // ...
];

// Create request object from $_GET
$request = (new RequestFactory($_GET))->create();

// Order data
if ($order = $request->getOrder()->current()) {
    usort($data, function ($a, $b) use ($request, $order) {
        $column = $request->getColumnAt($order->getColumn());
        $multiplier = ($order->getDir() === 'asc') ? 1 : -1;
        return strcmp($a[$column->getData()], $b[$column->getData()]) * $multiplier;
    });
}

// Response
header('Content-Type: application/json');

echo json_encode(
    new Response(
        $data,
        count($data),
        count($data),
        $request->getDraw()
    )
);
```

## PDO

```bash
cd examples/pdo
php -S localhost:8000

# Open browser http://localhost:8000/
```

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';

use Webinv\Datatables\RequestFactory;
use Webinv\Datatables\Response;

$pdo = new PDO('sqlite:mysqlite.db');

// Create data if not exists
if (0 === filesize('mysqlite.db')) {
    $sql = <<<'SQL'
CREATE TABLE IF NOT EXISTS tasks (
    id        INTEGER PRIMARY KEY,
    name      TEXT    NOT NULL,
    created_at TEXT
);

INSERT INTO tasks (id, name, created_at) VALUES
    (1, 'Task foo', date('now')),
    (2, 'Task bar', date('now')),
    (3, 'Task baz', date('now')),
    (4, 'Task 4', date('now')),
    (5, 'Task 5', date('now')),
    (6, 'Task 6', date('now')),
    (7, 'Task 7', date('now')),
    (8, 'Task 8', date('now')),
    (9, 'Task 9', date('now')),
    (10, 'Task 10', date('now')),
    (11, 'Task 11', date('now')),
    (12, 'Task 12', date('now'))
 ;
SQL;
    $pdo->exec($sql);
}

// Get request
$request = RequestFactory::fromGlobals();

// Order by
if ($order = $request->getOrder()->current()) {
    $columnName = $request->getColumnAt($order->getColumn());

    $column = (in_array($columnName, ['id', 'name', 'created_at'])) ? $columnName : 'id';
    $dir = ($order->getDir() === 'asc') ? 'ASC' : 'DESC';
} else {
    $column = 'id';
    $dir = 'ASC';
}

// Search
if ($search = $request->getSearch()->getValue()) {
    $sql = sprintf('SELECT * FROM tasks WHERE name LIKE :search ORDER BY %s %s LIMIT %d, %d',
        $column,
        $dir,
        $request->getStart(),
        $request->getLength()
    );

    $stmt = $pdo->prepare($sql);
    $stmt->execute([':search' => '%'.$search.'%']);

    $records = $stmt->fetchAll();

    $sql2 = 'SELECT COUNT(*) FROM tasks WHERE name LIKE :search';
    $stmt2 = $pdo->prepare($sql2);
    $stmt2->execute([':search' => '%'.$search.'%']);

    $total = $stmt2->fetchColumn();
} else {
    $sql = sprintf('SELECT * FROM tasks ORDER BY %s %s LIMIT %d, %d',
        $column,
        $dir,
        $request->getStart(),
        $request->getLength()
    );

    $records = $pdo->query($sql)->fetchAll();
    $total = $pdo->query('SELECT COUNT(*) FROM tasks')->fetchColumn();
}

// Response
header('Content-Type: application/json');

echo json_encode(
    new Response(
        $records,
        $total,
        $total,
        $request->getDraw()
    )
);
```