# API Resources Using Laravel 5.6

## What are API Resources
Presents a way to easily transform models into JSON responses.
API resources is made of:

* Resource Class : represents a single model that needs to be transformed to JSON.

* Resouce Collection : transforms collections of models into a JSON structure.

## User authentication
### Install
`$ composer require tymon/jwt-auth "1.0.*"`

### Publish Package's config file
`$ php artisan vendor:publish --provider="Tymon\JWTAuth\Providers\LaravelServiceProvider"`

This wil create a config/jwt.php file that will allow to configure the basics of the package.

### Generate Secret Key
`$ php artisan jwt:secret`

This will update the .env file with something like `JWT_SECERT=some_random_key`. This key will be used to sign our tokens.

### Update User.php Model
Before we can start to use the jwt-auth package, we need to update our User model to implement the
Tymon\JWTAuth\Contracts\JWTSubject contract as below:
```
//app/User.php
use Tymon\JWTAuth\Contracts\JWTSubject;

class User extends Authenticable implements JWTSubject
{
	...

	   public function getJWTIdentifier()
   	{
   		return $this->getKey();
   	}

   	public function getJWTCustomClaims()
   	{
   		return [];
   	}
}
```

### Configure the config/auth.php
Setting the api guar to use the jwt driver, and setting the api guard as the default.
```
// config.auth.php
'defaults' => [
      'guard' => 'api',
      'passwords' => 'users',
    ],

    ...

    'guards' => [
      'api' => [
        'driver' => 'jwt',
        'provider' => 'users',
      ],
    ],
```

### Create new auth controller
Create the controller file
`$ php artisan make:controller AuthController`

Then add the code below
```
// app/Http/Controllers/AuthController.php

// remember to add this to the top of the file
use App\User;

    public function register(Request $request)
    {
      $user = User::create([
        'name' => $request->name,
        'email' => $request->email,
        'password' => bcrypt($request->password),
      ]);

      $token = auth()->login($user);

      return $this->respondWithToken($token);
    }

    public function login(Request $request)
    {
      $credentials = $request->only(['email', 'password']);

      if (!$token = auth()->attempt($credentials)) {
        return response()->json(['error' => 'Unauthorized'], 401);
      }

      return $this->respondWithToken($token);
    }

    protected function respondWithToken($token)
    {
      return response()->json([
        'access_token' => $token,
        'token_type' => 'bearer',
        'expires_in' => auth()->factory()->getTTL() * 60
      ]);
    }
```

### Create the book resource
By default, resources will be placed in the `app/Http/Resources` directory of our application.
`$ php artisan make:resource BookResource`

Once that is created, add the code below on `// app/Http/Resources/BookResource.php`
```
    // app/Http/Resources/BookResource.php

    public function toArray($request)
    {
      return [
        'id' => $this->id,
        'title' => $this->title,
        'description' => $this->description,
        'created_at' => (string) $this->created_at,
        'updated_at' => (string) $this->updated_at,
        'user' => $this->user,
        'ratings' => $this->ratings,
      ];
    }
```

### Create the book controller
`$ php artisan make:controller BookController --api`

Then add the code below:

```
// app/Http/Controllers/BookController.php

    // add these at the top of the file
    use App\Book;
    use App\Http\Resources\BookResource;

    public function index()
    {
      return BookResource::collection(Book::with('ratings')->paginate(25));
    }

    public function store(Request $request)
    {
      $book = Book::create([
        'user_id' => $request->user()->id,
        'title' => $request->title,
        'description' => $request->description,
      ]);

      return new BookResource($book);
    }

    public function show(Book $book)
    {
      return new BookResource($book);
    }

    public function update(Request $request, Book $book)
    {
      // check if currently authenticated user is the owner of the book
      if ($request->user()->id !== $book->user_id) {
        return response()->json(['error' => 'You can only edit your own books.'], 403);
      }

      $book->update($request->only(['title', 'description']));

      return new BookResource($book);
    }

    public function destroy(Book $book)
    {
      $book->delete();

      return response()->json(null, 204);
    }
```

### Create Rating resource
* Create the Rating Resource
`$ php artisan make:resource RatingResource`

* Edit RatingResource.php
```
    // app/Http/Resources/RatingResource.php

    public function toArray($request)
    {
      return [
        'user_id' => $this->user_id,
        'book_id' => $this->book_id,
        'rating' => $this->rating,
        'created_at' => (string) $this->created_at,
        'updated_at' => (string) $this->updated_at,
        'book' => $this->book,
      ];
    }
```

* Creating rating controller
`$ php artisan make:controller RatingController`

* Edit the RatingController.php
```
    // app/Http/Controllers/RatingController.php

    // add these at the top of the file
    use App\Book;
    use App\Rating;
    use App\Http\Resources\RatingResource;

    public function store(Request $request, Book $book)
    {
      $rating = Rating::firstOrCreate(
        [
          'user_id' => $request->user()->id,
          'book_id' => $book->id,
        ],
        ['rating' => $request->rating]
      );

      return new RatingResource($rating);
    }
```