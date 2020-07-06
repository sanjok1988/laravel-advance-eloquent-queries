# laravel-advance-eloquent-queries


**1.  nested with function in eloquent:**
    case: user has many posts and post has many comments. we need user's posts with comments:
**Eloquent query:**  
 
    Users::with('posts')->get();

**models and relationships:**
 
**User model:**

    public function posts() {    
	    $this->hasMany('App\Post');    
    }

**Post Model:**
 

    protected $with="comments"; //it is important
      
    public function comments() {    
	    $this->hasMany('App\Comments');    
    }

> *Note: if you don't want global with comments in task (that means removing protected $with="comments"; from post model ) then the query
> becomes:*

 

    App\User::with('posts', 'posts.comments')->get();

if you want to select only some column then the code will be:

> *note: you must include foreign key in select clause*.
   

    return App\User::select('id', 'name')->with(['posts'=>function ($q) {    
	    $q->select('id', 'title', 'description', 'user_id'); //fk user_id    
    }], ['posts.comments'=>function ($q) {    
	    $q->select('id', 'post_id'); //fk post_id    
    }])->get();

  

//furthermore if you want user profile also then the code will be like:

    return App\User::select('id', 'name')    
    ->with(    
        ['posts'=>function ($q) {    
            $q->select('id', 'title', 'description', 'user_id');    
        }, 'profile'=>function ($q) {    
            $q->select('id', 'user_id');    
        }],    
        ['posts.comments'=>function ($q) {    
            $q->select('id', 'post_id');    
        }]    
    )    
    ->get();
 
OR,

    return App\User::select('id', 'name')    
	    ->with(['tasks'=>function ($q) {    
		    $q->select('id', 'title', 'description', 'user_id');	    
	    }], ['posts.comments'=>function ($q) {	    
		    $q->select('id', 'post_id');	    
	    }])	    
	    ->with(['profile'=>function ($q) {	    
		    $q->select('id', 'user_id');	    
	    }])->get();


**2.  Here a user has a location with many to many positions. This query finds users having position id 4 and location id 6. the tables are users, profile, locations. and profile has location_id.**
    
    User::whereHas('positions', function ($q) {    
	    $q->wherePositionId(4);    
    })->whereHas('profile', function ($q) {    
	    $q->whereLocationId(6);    
    })->with(['profile', 'positions'=>function ($q) {    
	    $q->wherePositionId(4);    
    }])->get();

//in user model

    public function profile()    
    {    
	    return $this->hasOne(Profile::class, 'user_id', 'id');    
    }

    public function positions()    
    {
        return $this->belongsToMany(\App\Positions::class, 'user_positions', 'user_id','position_id')    
            ->withPivot(['position_id', 'user_id']); //if you don't need pivot you can remove it
    
    }

  

**3.  The main difference is which side of the relationship holds the relationship's foreign key. The model that calls $this->belongsTo() is the owned model in one-to-one and many-to-one relationships and holds the key to the owning model.**   

//Example one-to-one relationship:

  

    class User extends Model {
    
	    public function car() {	    
		    // user has at maximum one car,	    
		    // so $user->car will return a single model	    
		    return $this->hasOne('Car');	    
	    }    
    }


    class Car extends Model {    
	    public function owner() {	    
		    // cars table has owner_id field that stores id of related user model	    
		    return $this->belongsTo('User');	    
	    }    
    }


//Example one-to-many relationship:  

    class User extends Model {    
	    public function phoneNumbers() {	    
		    // user can have multiple phone numbers,		    
		    // so $user->phoneNumbers will return a collection of models		    
		    return $this->hasMany('PhoneNumber');	    
	    }    
    }

    class PhoneNumber extends Model {    
	    public function owner() {	    
		    // phone_numbers table has owner_id field that stores id of related user model	    
		    return $this->belongsTo('User');	    
	    }    
    }

  

**4.  search query from scope in laravel model**
    
 

     public function scopeSearchResults($query)    
        {    
    	    return $query->when(!empty(request()->input('location', 0)), function($query) {    	    
	    	    $query->whereHas('location', function($query) {	    	    
		    	    $query->whereId(request()->input('location'));    	    
    	    });        
        })        
        ->when(!empty(request()->input('category', 0)), function($query) {        
	        $query->whereHas('categories', function($query) {	        
		        $query->whereId(request()->input('category'));        
	        });        
        })        
        ->when(!empty(request()->input('search', '')), function($query) {
        
	        $query->where(function($query) {	        
				$search = request()->input('search');	        
		        $query->where('title', 'LIKE', "%$search%")		        
		        ->orWhere('short_description', 'LIKE', "%$search%")		        
		        ->orWhere('full_description', 'LIKE', "%$search%")		        
		        ->orWhere('job_nature', 'LIKE', "%$search%")		        
		        ->orWhere('requirements', 'LIKE', "%$search%")		        
		        ->orWhere('address', 'LIKE', "%$search%")		        
		        ->orWhereHas('company', function($query) use($search) {		        
			        $query->where('name', 'LIKE', "%$search%");
		        });        
	        });        
        });        
      }
        
  

**5.  wrong and right way to write eloquent**    

What if you have and-or mix in your SQL query, like this:

  

    ... WHERE (gender = 'Male' and age >= 18) or (gender = 'Female' and age >= 65)

How to translate it into Eloquent? This is the wrong way:
 
    $q->where('gender', 'Male');    
    $q->orWhere('age', '>=', 18);    
    $q->where('gender', 'Female');    
    $q->orWhere('age', '>=', 65);

The order will be incorrect. The right way is a little more complicated, using closure functions as sub-queries:

  

    $q->where(function ($query) {    
	    $query->where('gender', 'Male')	    
	        ->where('age', '>=', 18);	    
	 })->orWhere(function($query) {	    
	    $query->where('gender', 'Female')	    
	        ->where('age', '>=', 65);    
    })

   
****6.  if you want to use ORM eloquent to get a query with the third table and select only particular attributes then here is the trick. Here users can have multiple positions in a company and each user has their own user profile. *note that user_id must be here to fetch data with a specific column. without foreign key eloquent cannot map relational table****
    
    $user = User::select("id", "name")    
	    ->with(['positions' => function ($query) {	    
		    $query->select('name');    
	    }, 'profile' => function ($query) {    
		    $query->select("user_id", "company_name");    
    }])->get();

  
In User model write many to many relation with user positions (designation)
  
    public function positions()
    {
    	return $this->belongsToMany(\App\Position::class, 'user_position', 'user_id', 'position_id')
    	->withPivot(['position_id', 'user_id']); //if you don't need pivot you can remove it
    }

  
In user Model relation with profile tablep

    public function profile()    
    {    
	    return $this->hasOne(Profile::class, 'user_id', 'id');    
    }

  

**7.  Eloquent : many to many relation between category and postsHere posts and categories have many to many relationships.**    

**# the tricks are to fetch posts having a particular category attribute with categories details.**

  //To get all posts with related category in each post  
      
    
    $career = Post::whereHas('categories', function ($q) {    
	    $q->whereSlug('career');    
    })->wherePostType('post')
    ->with(['categories'=>function ($q) {    
	    $q->whereSlug('career');    
    }])->get();
    
    //To get a category with all related posts
              
    $carrer = Category::with('posts')->whereSlug('career')->get();    
        
    //Relation in Category Model
     
    public function posts()
    {
	    return $this->belongsToMany('App\Post', 'category_post', 'category_id', 'post_id');    
    }
    
    //Relation in Post Model  
    
    public function categories()    
    {    
	    return $this->belongsToMany('App\Category', 'category_post', 'post_id', 'category_id');
    }
     

**8.  Nested WhereHas**
     

	$models = modelC::whereHas('modelB', function ($query) {
		$query->whereHas('modelA', function ($query) {
			$query->where('owner', 123);
		});
	})->get();




