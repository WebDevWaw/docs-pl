# Bazy danych: Konstruktor zapytań

- [Wprowadzenie](#introduction)
- [Pobieranie wyników](#retrieving-results)
    - [Porcjowanie wyników](#chunking-results)
    - [Agregacje](#aggregates)
- [Selects](#selects)
- [Wyrażenia modyfikujące](#raw-expressions)
- [Łączenie (Join)](#joins)
- [Scalanie (Union)](#unions)
- [Klauzule ograniczajace (Where)](#where-clauses)
    - [Grupowanie Parametrów](#parameter-grouping)
    - [Where Exists Clauses](#where-exists-clauses)
    - [JSON Where Clauses](#json-where-clauses)
- [Sortowanie, Grupowanie, Limity i Offsety](#ordering-grouping-limit-and-offset)
- [Klauzule Warunkowe](#conditional-clauses)
- [Dodanie (Insert)](#inserts)
- [Aktualizacja (Updates)](#updates)
    - [Updating JSON Columns](#updating-json-columns)
    - [Increment & Decrement](#increment-and-decrement)
- [Usuwanie (Delete)](#deletes)
- [Pessimistic Locking](#pessimistic-locking)

<a name="introduction"></a>
## Wprowadzenie

Narzędzie do tworzenia zapytań baz danych w Laravel zapewnia wygodny, płynny interfejs do tworzenia i uruchamiania zapytań bazodanowych. Może być on użyty  wykonywania większości operacji bazodanowych w twojej aplikacji i współdziała ze wszystkimi rodzajami baz.

Konstruktor zapytań w Laravel używa PDO parameter binding do zabezpieczania twojej aplikacji przed atakami SQL injection. Nie ma potrzeby dodatkowego filtrowania ciągów znaków przekazywanych jako  wiązania.

<a name="retrieving-results"></a>
## Pobieranie wyników

#### Pobieranie Wszystkich Wierszy Z Tabeli

Możesz użyć metody `table` na fasadzie `DB` do stworzenia początkowego zapytania. Metoda `table` method zwraca instancję konstruktora zapytań dla danej tabeli, pozwalajać dowiązać ci więcej elementów precyzujących zapytanie, aby finalnie uzyskac wynik przy użyciu metody `get`:

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Support\Facades\DB;
    use App\Http\Controllers\Controller;

    class UserController extends Controller
    {
        /**
         * Show a list of all of the application's users.
         *
         * @return Response
         */
        public function index()
        {
            $users = DB::table('users')->get();

            return view('user.index', ['users' => $users]);
        }
    }

Metoda `get` zwracana przez  `Illuminate\Support\Collection` zawiera wyniki, w których każdy jeden jest instancją obiektu PHP `StdClass`. Możesz mieć  dostęp do wartości każdej  kolumny, przez wskazanie kolumny jako właściwości obiektu:

    foreach ($users as $user) {
        echo $user->name;
    }

#### Pobieranie Pojedynczego Wiesza / Kolumny z Tabeli

Jeżeli zwyczajnie potrzbujesz otrzymać pojedynczy wiesz danych z tabeli, możesz użyć metody `first`. Ta metoda zwróci pojedynczy obiekt `StdClass`:

    $user = DB::table('users')->where('name', 'John')->first();

    echo $user->name;

Jeśli nawet nie potrzebujesz całego wiersza, możesz wyodrębnić pojedynczą wartość z rekordu przy użyciu metody `value`. Metoda ta zwróci bezpośrednio wartość wskazanej kolumny:

    $email = DB::table('users')->where('name', 'John')->value('email');

#### Pobieranie Listy Wartości Danej Kolumny

Jeśli chciałbyś pobrać kolekcję zawierającą wszystkie wartości jednej kolumny, możesz użyć metody `pluck`. W tym przykładzie otrzymamy kolekcję tytułów z wskazanej tabeli:

    $titles = DB::table('roles')->pluck('title');

    foreach ($titles as $title) {
        echo $title;
    }

Możesz rónież wskazać własną listę nazw kolumn, które mają być zwrócone w kolekcji:

    $roles = DB::table('roles')->pluck('title', 'name');

    foreach ($roles as $name => $title) {
        echo $title;
    }

<a name="chunking-results"></a>
### Porcjowanie Wyników

Jeśli pracujesz z tysiącami rekordów danych, rozważ użycie metody `chunk`. Ta metoda pobiera każdorazowo  małą porcję wyników i wprwadza każdą część do  `Closure` w celu dalszego przetwarzania. Ta metoda jest bardzo przydatna podczas korzystania z [komend Artisan ](/docs/{{version}}/artisan) kiedy przetwarzasz tysiace rekordów. Na przykład, spróbujmy przetworzyć całość tabeli `users` w porcjach po 100 rekordów:

    DB::table('users')->orderBy('id')->chunk(100, function ($users) {
        foreach ($users as $user) {
            //
        }
    });

Możesz zatrzymac przetwarzanie kolejnych partii zwracajac `false` w obrzarze `Closure`:

    DB::table('users')->orderBy('id')->chunk(100, function ($users) {
        // Process the records...

        return false;
    });

<a name="aggregates"></a>
### Agregacje

Narzędzie budowania zapytań zawiera również wiele różnych metod agregujących, takich jak `count`, `max`, `min`, `avg`, i `sum`. You may call any of these methods after constructing your query:
Możesz odwoływać sie do nich kolejno, zaraz po stworzeniu swojego zapytania:

    $users = DB::table('users')->count();

    $price = DB::table('orders')->max('price');

Oczywiście, możesz użyć kombinacji tych metod w połączeniu z innymi warunakmi:

    $price = DB::table('orders')
                    ->where('finalized', 1)
                    ->avg('price');

<a name="selects"></a>
## Selects

#### Specifying A Select Clause

Oczywiscie nie zawsze możesz chcieć pobierać wszytkie kolumny z tabeli w bazie. Używając metody `select`, możesz określać własne klauzule `select` dla swoich zapytań:

    $users = DB::table('users')->select('name', 'email as user_email')->get();

Metoda `distinct`pozwoli ci wymusić zapytanie, które zwróci odpowiednio przefiltrowane wyniki:

    $users = DB::table('users')->distinct()->get();

Jeśłi masz już instancję konstuktora zapytań i ższczysz sobie dodać kolejną kolumnę do obecnego zapytania, możesz użyć meody `addSelect`:

    $query = DB::table('users')->select('name');

    $users = $query->addSelect('age')->get();

<a name="raw-expressions"></a>
## Wyrażenia modyfikujące

Czasami możesz potrzebować zwodyfikować domyłśne zapytanie używając wyrażenia modyfikujacego. Takie wyrażenie może zostać wstrzyknięte do twojego zapytania jako ciąg znaków, zatem upewnij się czy nie tworzysz potencjalnego punktu dla SQL injection! Aby wykorzystać takie wyrażenie, skorzystaj z metody `DB::raw`:

    $users = DB::table('users')
                         ->select(DB::raw('count(*) as user_count, status'))
                         ->where('status', '<>', 1)
                         ->groupBy('status')
                         ->get();

<a name="joins"></a>
## Łączenie (Join)

#### Złączenia

The query builder may also be used to write join statements. To perform a basic "inner join", you may use the `join` method on a query builder instance. The first argument passed to the `join` method is the name of the table you need to join to, while the remaining arguments specify the column constraints for the join. Of course, as you can see, you can join to multiple tables in a single query:

    $users = DB::table('users')
                ->join('contacts', 'users.id', '=', 'contacts.user_id')
                ->join('orders', 'users.id', '=', 'orders.user_id')
                ->select('users.*', 'contacts.phone', 'orders.price')
                ->get();

#### Lewostronne Złączenia

If you would like to perform a "left join" instead of an "inner join", use the `leftJoin` method. The `leftJoin` method has the same signature as the `join` method:

    $users = DB::table('users')
                ->leftJoin('posts', 'users.id', '=', 'posts.user_id')
                ->get();

#### Krzyżowe Łączenie

To perform a "cross join" use the `crossJoin` method with the name of the table you wish to cross join to. Cross joins generate a cartesian product between the first table and the joined table:

    $users = DB::table('sizes')
                ->crossJoin('colours')
                ->get();

#### Zaawansowałe Łączenia

You may also specify more advanced join clauses. To get started, pass a `Closure` as the second argument into the `join` method. The `Closure` will receive a `JoinClause` object which allows you to specify constraints on the `join` clause:

    DB::table('users')
            ->join('contacts', function ($join) {
                $join->on('users.id', '=', 'contacts.user_id')->orOn(...);
            })
            ->get();

If you would like to use a "where" style clause on your joins, you may use the `where` and `orWhere` methods on a join. Instead of comparing two columns, these methods will compare the column against a value:

    DB::table('users')
            ->join('contacts', function ($join) {
                $join->on('users.id', '=', 'contacts.user_id')
                     ->where('contacts.user_id', '>', 5);
            })
            ->get();

<a name="unions"></a>
## Scalanie (Union)

Konstruktor zapytań zapewnia łatwy sposób scalenie  wyników "dwóch" zapytań. Na przykład możesz utworzyć zapytanie początkowe i użyć metody `union`, aby scalić ją z drugim zapytaniem:

    $first = DB::table('users')
                ->whereNull('first_name');

    $users = DB::table('users')
                ->whereNull('last_name')
                ->union($first)
                ->get();

> {Wskazówka} Metoda `unionAll` jest również dostępna i posiada taką samą sygnaturę jak `union`.

<a name="where-clauses"></a>
## Klauzule ograniczajace (Where)

#### Simple Where Clauses

You may use the `where` method on a query builder instance to add `where` clauses to the query. The most basic call to `where` requires three arguments. The first argument is the name of the column. The second argument is an operator, which can be any of the database's supported operators. Finally, the third argument is the value to evaluate against the column.

For example, here is a query that verifies the value of the "votes" column is equal to 100:

    $users = DB::table('users')->where('votes', '=', 100)->get();

For convenience, if you simply want to verify that a column is equal to a given value, you may pass the value directly as the second argument to the `where` method:

    $users = DB::table('users')->where('votes', 100)->get();

Of course, you may use a variety of other operators when writing a `where` clause:

    $users = DB::table('users')
                    ->where('votes', '>=', 100)
                    ->get();

    $users = DB::table('users')
                    ->where('votes', '<>', 100)
                    ->get();

    $users = DB::table('users')
                    ->where('name', 'like', 'T%')
                    ->get();

You may also pass an array of conditions to the `where` function:

    $users = DB::table('users')->where([
        ['status', '=', '1'],
        ['subscribed', '<>', '1'],
    ])->get();

#### Operator lub (or)

You may chain where constraints together as well as add `or` clauses to the query. The `orWhere` method accepts the same arguments as the `where` method:

    $users = DB::table('users')
                        ->where('votes', '>', 100)
                        ->orWhere('name', 'John')
                        ->get();

#### Dodatkowe kaluzule ograniczjące (where)

**whereBetween**

The `whereBetween` method verifies that a column's value is between two values:

    $users = DB::table('users')
                        ->whereBetween('votes', [1, 100])->get();

**whereNotBetween**

The `whereNotBetween` method verifies that a column's value lies outside of two values:

    $users = DB::table('users')
                        ->whereNotBetween('votes', [1, 100])
                        ->get();

**whereIn / whereNotIn**

The `whereIn` method verifies that a given column's value is contained within the given array:

    $users = DB::table('users')
                        ->whereIn('id', [1, 2, 3])
                        ->get();

The `whereNotIn` method verifies that the given column's value is **not** contained in the given array:

    $users = DB::table('users')
                        ->whereNotIn('id', [1, 2, 3])
                        ->get();

**whereNull / whereNotNull**

The `whereNull` method verifies that the value of the given column is `NULL`:

    $users = DB::table('users')
                        ->whereNull('updated_at')
                        ->get();

The `whereNotNull` method verifies that the column's value is not `NULL`:

    $users = DB::table('users')
                        ->whereNotNull('updated_at')
                        ->get();

**whereDate / whereMonth / whereDay / whereYear**

The `whereDate` method may be used to compare a column's value against a date:

    $users = DB::table('users')
                    ->whereDate('created_at', '2016-12-31')
                    ->get();

The `whereMonth` method may be used to compare a column's value against a specific month of a year:

    $users = DB::table('users')
                    ->whereMonth('created_at', '12')
                    ->get();

The `whereDay` method may be used to compare a column's value against a specific day of a month:

    $users = DB::table('users')
                    ->whereDay('created_at', '31')
                    ->get();

The `whereYear` method may be used to compare a column's value against a specific year:

    $users = DB::table('users')
                    ->whereYear('created_at', '2016')
                    ->get();

**whereColumn**

The `whereColumn` method may be used to verify that two columns are equal:

    $users = DB::table('users')
                    ->whereColumn('first_name', 'last_name')
                    ->get();

You may also pass a comparison operator to the method:

    $users = DB::table('users')
                    ->whereColumn('updated_at', '>', 'created_at')
                    ->get();

The `whereColumn` method can also be passed an array of multiple conditions. These conditions will be joined using the `and` operator:

    $users = DB::table('users')
                    ->whereColumn([
                        ['first_name', '=', 'last_name'],
                        ['updated_at', '>', 'created_at']
                    ])->get();

<a name="parameter-grouping"></a>
### Grupowanie Parametrów

Sometimes you may need to create more advanced where clauses such as "where exists" clauses or nested parameter groupings. The Laravel query builder can handle these as well. To get started, let's look at an example of grouping constraints within parenthesis:

    DB::table('users')
                ->where('name', '=', 'John')
                ->orWhere(function ($query) {
                    $query->where('votes', '>', 100)
                          ->where('title', '<>', 'Admin');
                })
                ->get();

As you can see, passing a `Closure` into the `orWhere` method instructs the query builder to begin a constraint group. The `Closure` will receive a query builder instance which you can use to set the constraints that should be contained within the parenthesis group. The example above will produce the following SQL:

    select * from users where name = 'John' or (votes > 100 and title <> 'Admin')

<a name="where-exists-clauses"></a>
### Where Exists Clauses

The `whereExists` method allows you to write `where exists` SQL clauses. The `whereExists` method accepts a `Closure` argument, which will receive a query builder instance allowing you to define the query that should be placed inside of the "exists" clause:

    DB::table('users')
                ->whereExists(function ($query) {
                    $query->select(DB::raw(1))
                          ->from('orders')
                          ->whereRaw('orders.user_id = users.id');
                })
                ->get();

The query above will produce the following SQL:

    select * from users
    where exists (
        select 1 from orders where orders.user_id = users.id
    )

<a name="json-where-clauses"></a>
### JSON Where Clauses

Laravel also supports querying JSON column types on databases that provide support for JSON column types. Currently, this includes MySQL 5.7 and PostgreSQL. To query a JSON column, use the `->` operator:

    $users = DB::table('users')
                    ->where('options->language', 'en')
                    ->get();

    $users = DB::table('users')
                    ->where('preferences->dining->meal', 'salad')
                    ->get();

<a name="ordering-grouping-limit-and-offset"></a>
## Sortowanie, Grupowanie, Limity i Offsety

#### orderBy

The `orderBy` method allows you to sort the result of the query by a given column. The first argument to the `orderBy` method should be the column you wish to sort by, while the second argument controls the direction of the sort and may be either `asc` or `desc`:

    $users = DB::table('users')
                    ->orderBy('name', 'desc')
                    ->get();

#### latest / oldest

The `latest` and `oldest` methods allow you to easily order results by date. By default, result will be ordered by the `created_at` column. Or, you may pass the column name that you wish to sort by:

    $user = DB::table('users')
                    ->latest()
                    ->first();

#### inRandomOrder

The `inRandomOrder` method may be used to sort the query results randomly. For example, you may use this method to fetch a random user:

    $randomUser = DB::table('users')
                    ->inRandomOrder()
                    ->first();

#### groupBy / having / havingRaw

The `groupBy` and `having` methods may be used to group the query results. The `having` method's signature is similar to that of the `where` method:

    $users = DB::table('users')
                    ->groupBy('account_id')
                    ->having('account_id', '>', 100)
                    ->get();

The `havingRaw` method may be used to set a raw string as the value of the `having` clause. For example, we can find all of the departments with sales greater than $2,500:

    $users = DB::table('orders')
                    ->select('department', DB::raw('SUM(price) as total_sales'))
                    ->groupBy('department')
                    ->havingRaw('SUM(price) > 2500')
                    ->get();

#### skip / take

To limit the number of results returned from the query, or to skip a given number of results in the query, you may use the `skip` and `take` methods:

    $users = DB::table('users')->skip(10)->take(5)->get();

Alternatively, you may use the `limit` and `offset` methods:

    $users = DB::table('users')
                    ->offset(10)
                    ->limit(5)
                    ->get();

<a name="conditional-clauses"></a>
## Klauzule Warunkowe

Sometimes you may want clauses to apply to a query only when something else is true. For instance you may only want to apply a `where` statement if a given input value is present on the incoming request. You may accomplish this using the `when` method:

    $role = $request->input('role');

    $users = DB::table('users')
                    ->when($role, function ($query) use ($role) {
                        return $query->where('role_id', $role);
                    })
                    ->get();


The `when` method only executes the given Closure when the first parameter is `true`. If the first parameter is `false`, the Closure will not be executed.

You may pass another Closure as the third parameter to the `when` method. This Closure will execute if the first parameter evaluates as `false`. To illustrate how this feature may be used, we will use it to configure the default sorting of a query:

    $sortBy = null;

    $users = DB::table('users')
                    ->when($sortBy, function ($query) use ($sortBy) {
                        return $query->orderBy($sortBy);
                    }, function ($query) {
                        return $query->orderBy('name');
                    })
                    ->get();


<a name="inserts"></a>
## Dodanie (Insert)

The query builder also provides an `insert` method for inserting records into the database table. The `insert` method accepts an array of column names and values:

    DB::table('users')->insert(
        ['email' => 'john@example.com', 'votes' => 0]
    );

You may even insert several records into the table with a single call to `insert` by passing an array of arrays. Each array represents a row to be inserted into the table:

    DB::table('users')->insert([
        ['email' => 'taylor@example.com', 'votes' => 0],
        ['email' => 'dayle@example.com', 'votes' => 0]
    ]);

#### Auto-Incrementing IDs

If the table has an auto-incrementing id, use the `insertGetId` method to insert a record and then retrieve the ID:

    $id = DB::table('users')->insertGetId(
        ['email' => 'john@example.com', 'votes' => 0]
    );

> {note} When using PostgreSQL the `insertGetId` method expects the auto-incrementing column to be named `id`. If you would like to retrieve the ID from a different "sequence", you may pass the sequence name as the second parameter to the `insertGetId` method.

<a name="updates"></a>
## Aktualizacja (Updates)

Of course, in addition to inserting records into the database, the query builder can also update existing records using the `update` method. The `update` method, like the `insert` method, accepts an array of column and value pairs containing the columns to be updated. You may constrain the `update` query using `where` clauses:

    DB::table('users')
                ->where('id', 1)
                ->update(['votes' => 1]);

<a name="updating-json-columns"></a>
### Updating JSON Columns

When updating a JSON column, you should use `->` syntax to access the appropriate key in the JSON object. This operation is only supported on databases that support JSON columns:

    DB::table('users')
                ->where('id', 1)
                ->update(['options->enabled' => true]);

<a name="increment-and-decrement"></a>
### Increment & Decrement

The query builder also provides convenient methods for incrementing or decrementing the value of a given column. This is simply a shortcut, providing a more expressive and terse interface compared to manually writing the `update` statement.

Both of these methods accept at least one argument: the column to modify. A second argument may optionally be passed to control the amount by which the column should be incremented or decremented:

    DB::table('users')->increment('votes');

    DB::table('users')->increment('votes', 5);

    DB::table('users')->decrement('votes');

    DB::table('users')->decrement('votes', 5);

You may also specify additional columns to update during the operation:

    DB::table('users')->increment('votes', 1, ['name' => 'John']);

<a name="deletes"></a>
## Usuwanie (Delete)

The query builder may also be used to delete records from the table via the `delete` method. You may constrain `delete` statements by adding `where` clauses before calling the `delete` method:

    DB::table('users')->delete();

    DB::table('users')->where('votes', '>', 100)->delete();

If you wish to truncate the entire table, which will remove all rows and reset the auto-incrementing ID to zero, you may use the `truncate` method:

    DB::table('users')->truncate();

<a name="pessimistic-locking"></a>
## Pessimistic Locking

The query builder also includes a few functions to help you do "pessimistic locking" on your `select` statements. To run the statement with a "shared lock", you may use the `sharedLock` method on a query. A shared lock prevents the selected rows from being modified until your transaction commits:

    DB::table('users')->where('votes', '>', 100)->sharedLock()->get();

Alternatively, you may use the `lockForUpdate` method. A "for update" lock prevents the rows from being modified or from being selected with another shared lock:

    DB::table('users')->where('votes', '>', 100)->lockForUpdate()->get();
