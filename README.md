# Alamofire, AlamofireObjectMapper and Realm demo project

This is a step-by-step walkthrough of my talk at **CocoaHeads Sthlm, April 2017**,
where I demonstrated how to use `Alamofire 4` and `AlamofireObjectMapper` to pull
and parse data from an API. I also used `Realm` for seamless database persistency
and a great library called `Dip` to manage dependencies in the app.

In this post, I'll recreate this app from scratch, with some modifications. Since
the Yelp API doesn't let you persist data you pull out from it, I will modify the
setup slightly, to use a static API instead. Read more on this a bit further down.


# Video

You can watch the original presentation [here](https://www.youtube.com/watch?v=LuKehlKoN7o&lc=z22qu35a4xawiriehacdp435fnpjgmq2f54mjmyhi2tw03c010c.1502618893412377).
However, it focuses more on concepts than code, but could be a nice complement to
this more code-oriented post.


# Source Code

I really recommend you to create a blank iOS project and go through all the steps
in this post yourself. However, you can download the complete source code for the
app from [this GitHub repository](https://github.com/danielsaidi/CocoaHeads-2017-04-03-Alamofire-Realm).

If you want to run the demo application in the repo, you have to run `pod install`
from the `DemoApplication` folder, then use `DemoApplication.xcworkspace` instead
of `DemoApplication.xcodeproj`. 


# Prerequisites

For this tutorial, you'll need to download and install [Xcode](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=0ahUKEwi7lP7s--XVAhVmEpoKHcVkBzUQ0EMIKg&url=https%3A%2F%2Fdeveloper.apple.com%2Fxcode%2Fdownloads%2F&usg=AFQjCNFpOFz2CXarfnUzyEM1Lbia_k7fZw)
from Apple Developer Portal. You also need to [CocoaPods](https://cocoapods.org/),
which is used to manage external dependencies in your app.


# Disclaimer 

In large app projects, I prefer to extract as much code and logic as possible to
separate libraries, which I then use as decoupled building blocks. For instance,
I would keep domain logic in a domain library, which doesn't know anything about
the app. In this small project, however, I will keep the domain model in the app.

I use to separate public and private functions and any interface implementations
into separate extensions as well, but will skip that pattern in this demo, so we
get as little code and conventions as possible. 


# Why use a static API?

In this demo, we will use a static API to fetch movies in two different ways. The
API is just a static Jekyll site with a small movie collection, that lets us grab
top grossing and top rated movies from 2016.

This may seem limited, but will hopefully mean that we can focus more time on the
specific features of Alamofire and Realm instead of understanding an external API
and having to set up a developer account, get authorization up and working etc. I 
hope you can look past these limitations, as you take your newfound knowledge out
into the real world.

Ready? Let's go!


# Step 1 - Create a new iOS project

We'll start by creating a new iOS project in `Xcode`. Open Xcode, click `Create a
new Xcode project`, select the `iOS` tab, then select a `Single View Application`.
In this tutorial, I'll name the project `DemoApplication`:

![Image](img/1-create.png)

Press `Next` to select where to create your project. You will then have a new iOS
project just waiting to be filled with amazing code.


# Step 2 - Describe the domain model

In this demo, we will fetch movies from a fake API. A `Movie` has some basic info
and a `cast` property that contains `Actor` objects. For simplicity, `Actor` only
has a name and is used to demonstrate recursive mapping.
 
To avoid having an app that is coupled to a specific implementation of the domain
model, let us start by defining this model as protocols. Create a `Domain` folder
in the project root, then add a `Model` sub folder to it and add these two files:

```
// Movie.swift

import Foundation

protocol Movie {
    
    var name: String { get }
    var releaseDate: Date { get }
    var grossing: Int { get }
    var rating: Double { get }
    var cast: [Actor] { get }
}
```

```
// Actor.swift

import Foundation

protocol Actor {
    
    var name: String { get }
}
```

As you will see later on, our app will only care about protocols and not specific
implementations. This makes it super easy to switch out which implementations the
app should use. Stay tuned.


# Step 3 - Describe the domain logic

Since we have a static API with no client or user authorization or authentication,
we don't have to start with this. Instead, we can describe how we want the app to
fetch movies.

Add a `Services` sub folder to `Domain` and add this file:

```
// MovieService.swift

import Foundation

typealias MovieResult = (_ movie: Movie?, _ error: Error?) -> ()
typealias MoviesResult = (_ movies: [Movie], _ error: Error?) -> ()


protocol MovieService: class {
    
    func getMovie(id: Int, completion: @escaping MovieResult)
    func getTopGrossingMovies(year: Int, completion: @escaping MoviesResult)
    func getTopRatedMovies(year: Int, completion: @escaping MoviesResult)
}

```

It took me a while to start using `typealias`, but I really like it, and it does
greatly simplify describing async results.

This service tells us that we will be able to get movies asynchronously (well, a
sompletion block implies it, but doesn't *require* the operation to be async) in
two ways. The result data for the two operations will be indentically formatted. 


# Step 4 - Add Alamofire and AlamofireObjectMapper to the project

Before we can add an API-specific implementation to the project, we must add two
so called `pods` to the project. Make sure CocoaPods is properly installed, then
run the following command from the `DemoApplication` folder:

```
pod init
```

This will create a `pod file` in your project root, in which you can specify the
pods the app requires. Make sure that the file looks like this:

```
// Podfile

use_frameworks!

target 'DemoApplication' do
    pod 'Alamofire', '~> 4.0.0'
    pod 'AlamofireObjectMapper', '~> 4.0' 
end
``` 

Now, close this file and run the following Terminal command from the same folder:

```
pod install
```

This makes CocoaPods download the pods and link them to your Xcode project. Once
that is done, close the project, then open `DemoApplication.xcworkspace` instead.


# Step 5 - Create an API-specific domain model

We are now ready to use Alamofire to pull some data from our API. Have a look at
the (very limited) data that can be fetched from it:

* [Single movie (by id)](http://danielsaidi.com/CocoaHeads-2017-04-03-Alamofire-Realm/api/movies/1)
* [Top 10 Grossing Movies 2016](http://danielsaidi.com/CocoaHeads-2017-04-03-Alamofire-Realm/api/movies/topGrossing/2016)
* [Top 10 Rated Movies 2016](http://danielsaidi.com/CocoaHeads-2017-04-03-Alamofire-Realm/api/movies/topRated/2016)

Let's create an API-specific implementation of the domain model. Create an `API`
folder in the project root, next to `Domain`, then add a `Model` sub folder and
add these two files:

```
// ApiMovie.swift

import ObjectMapper

class ApiMovie: NSObject, Movie, Mappable {
    
    required public init?(map: Map) {
        super.init()
    }
    
    var name = ""
    var releaseDate = Date(timeIntervalSince1970: 0)
    var grossing = 0
    var rating = 0.0
    fileprivate var _cast = [ApiActor]()
    var cast = [Actor]()
    
    func mapping(map: Map) {
        name <- map["name"]
        releaseDate <- (map["releaseDate"], DateTransform())
        grossing <- map["grossing"]
        rating <- map["grossing"]
        _cast <- map["cast"]
        cast = _cast
    }
}
``` 

```
// ApiActor.swift

import ObjectMapper

class ApiActor: NSObject, Actor, Mappable {
    
    required public init?(map: Map) {
        super.init()
    }
    
    var name = ""
    
    func mapping(map: Map) {
        name <- map["name"]
    }
}
``` 

As you can see, these classes implement their respecive model protocol, then add
ObjectMapper mapping ontop. While `ApiActor` is straightforward, some things can
be said about `ApiMovie`:

* The release date is parsed using a DateTransform() instance. We may have to do
tweak this later on.

* The `Movie` protocol has an [Actor] array, but the mapping requires [ApiActor].
We therefore use a fileprivate `_cast` property for mappin, then copy the mapped
data to the public `cast` property.

If we have set things up properly, we should now be able to point Alamofire to a
certain api url and recursively parse movie data without any effort.


# Step 6 - Setting the API foundation

Before we create an API-specific `MovieService` implementation, let's setup some
utils in the `API` folder.

## Managing API environments

Since we developers often have to switch between different API environments (e.g.
test and production) I use to have an enum where I manage available environments.
We only have one environment in this case, but let's do it anyway:

```
// ApiEnvironment.swift

import Foundation

enum ApiEnvironment: String { case
    
    production = "http://danielsaidi.com/CocoaHeads-2017-04-03-Alamofire-Realm/api/"
    
    var url: String {
        return rawValue
    }
}

```

## Managing API routes

With the API environment info in place, we can now list all static API routes of
interest in another enum:

```
// ApiRoute.swift

enum ApiRoute { case
    
    movie(id: Int),
    topGrossingMovies(year: Int),
    topRatedMovies(year: Int)
    
    var path: String {
        switch self {
        case .movie(let id): return "movies/\(id)"
        case .topGrossingMovies(let year): return "movies/topGrossing/\(year)"
        case .topRatedMovies(let year): return "movies/topRated/\(year)"
        }
    }
    
    func url(for environment: ApiEnvironment) -> String {
        return "\(environment.url)/\(path)"
    }
}
```

Since `year` is a dynamic part of the API movie paths, we add a year argument to
the movie enum cases as well.

## Managing API context

I use to have an API context that holds all API-specific information for the app,
such as target environment and auth tokens. If you use a context singleton in an
app, it is super easy to change its information and have it automatically affect
all API-based services in the app.

Let's create an `ApiContext` protocol, as well as a non-persisted implementation,
in a new `Context` sub folder in the `Api` folder:

```
// ApiContext.swift

import Foundation

protocol ApiContext: class {
    
    var environment: ApiEnvironment { get set }
}
```   

```
// NonPersistedApiContext.swift

import Foundation

class NonPersistentApiContext: ApiContext {
    
    init(environment: ApiEnvironment) {
        self.environment = environment
    }
    
    var environment: ApiEnvironment
}
```

We can now inject this context into all out API-specific service implementations,
as you will see later.

## Specifying basic API behavior

To simplify how to talk with the API using Alamofire, let us create a base class
for our API-based services. Add a `Services` sub folder to the `Api` folder then
add this file to it:

```
// AlamofireService.swift

class AlamofireService {    
    
    init(context: ApiContext) {
        self.context = context
    }
    
    
    var context: ApiContext
    
    
    func get(at route: ApiRoute, params: Parameters? = nil) -> DataRequest {
        return request(at: route, method: .get, params: params, encoding: URLEncoding.default)
    }
    
    func post(at route: ApiRoute, params: Parameters? = nil) -> DataRequest {
        return request(at: route, method: .post, params: params, encoding: JSONEncoding.default)
    }
    
    func put(at route: ApiRoute, params: Parameters? = nil) -> DataRequest {
        return request(at: route, method: .put, params: params, encoding: JSONEncoding.default)
    }
    
    func request(at route: ApiRoute, method: HTTPMethod, params: Parameters?, encoding: ParameterEncoding) -> DataRequest {
        let url = route.url(for: context.environment)
        return Alamofire.request(url, method: method, parameters: params, encoding: encoding).validate()
    }
}
``` 

Restricting our services to requests that are specified in `ApiRoute` will ensure
that the app doesn't make any unspecified requests, which increases the stability,
and testability of the app. In you really need to call custom URLs, I suggest you
add a `.custom(...)` case in the `ApiRoute` enum. 

Ok, that was a nice long preparation phase mainly aimed at setting up a good base
for the future. I think we're now ready to fetch some movies from the API.


# Step 7 - Creating an API-specific movie service

Now, finally, let's create an API-specific service and pull some data out of that
API, shall we? In the `Api/Services` folder, add this file:

```
import Alamofire
import AlamofireObjectMapper

class AlamofireMovieService: AlamofireService, MovieService {
    
    func getMovie(id: Int, completion: @escaping MovieResult) {
        get(at: .movie(id: id)).responseObject {
            (res: DataResponse<ApiMovie>) in
            completion(res.result.value, res.result.error)
        }
    }
    
    func getTopGrossingMovies(year: Int, completion: @escaping MoviesResult) {
        get(at: .topGrossingMovies(year: year)).responseArray {
            (res: DataResponse<[ApiMovie]>) in
            completion(res.result.value ?? [], res.result.error)
        }
    }
    
    func getTopRatedMovies(year: Int, completion: @escaping MoviesResult) {
        get(at: .topRatedMovies(year: year)).responseArray {
            (res: DataResponse<[ApiMovie]>) in
            completion(res.result.value ?? [], res.result.error)
        }
    }
}
```

As you can see, the implementation is super-simple. Just perform get requests on
the routes and specify the return type, then Alamofire and AlamofireObjectMapper
will automatically take care of fetching and mapping the returned data.

Note that `getMovie` uses `responseObject` to specify return type, while the two
other functions use `responseArray`. This is due to the fact that a single movie
is returned as an object while top grossing and top rated movies are returned as
an array. If these two arrays would be returned in an embedded object instead (a
very common case), you would have to specify a new API-specific mapping type and
use `responseObject` instead.


# Step 8 - Perform your very first request

We will now setup the demo application and use it to perform our very first data
request from the API.

First, remove all boilerplate code from `AppDelegate` and `ViewController`. Keep
`viewDidLoad` and add the following code to it:

```

```

**IMPORTANT** Before you test your app, you have to allow it to perform requests.
Add the following to `Info.plist`:

```
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

In real-life apps, you should specify exactly which domains you may requests and
not allow arbitrary loads, but in this demo app we will live a little dangerous.

Now run the app, which should print the following:

```

```