# Oxygen.jl

<!-- START HTML -->
<div>
  </br>
  <p align="center"><img src="oxygen.png" width="20%"></p>
  <p align="center">
    <strong>A breath of fresh air for programming web apps in Julia.</strong>
  </p>
  <p align="center">
    <a href='https://juliahub.com/ui/Packages/Oxygen/JtS3f'><img src='https://juliahub.com/docs/Oxygen/version.svg?token=7GV8X1C98M' alt='Version' /></a>
    <a href='https://ndortega.github.io/Oxygen.jl/stable/'><img src='https://img.shields.io/badge/docs-stable-blue.svg' alt='documentation stable' /></a>
    <a href='https://github.com/ndortega/Oxygen.jl/actions/workflows/ci.yml'><img src='https://github.com/ndortega/Oxygen.jl/actions/workflows/ci.yml/badge.svg' alt='Build Status' /></a>
    <a href='https://codecov.io/gh/ndortega/Oxygen.jl'><img src='https://codecov.io/gh/ndortega/Oxygen.jl/branch/master/graph/badge.svg?token=7GV8X1C98M' alt='Coverage Status' /></a>
  </p>
</div>
<!-- END HTML -->

## About
Oxygen is a micro-framework built on top of the HTTP.jl library. 
Breathe easy knowing you can quickly spin up a web server with abstractions you're already familiar with.

## Features

- Straightforward routing (`@get`, `@post`, `@put`, `@patch`, `@delete` and `@route` macros)
- Auto-generated swagger documentation
- Out-of-the-box JSON serialization & deserialization (customizable)
- Type definition support for path parameters
- Built-in multithreading support
- Static file hosting
- Middleware chaining (at the application, router, and route levels)
- Route tagging
- Repeat tasks

## Installation

```julia
pkg> add Oxygen
```

## Minimalistic Example

Create a web-server with very few lines of code
```julia
using Oxygen
using HTTP

@get "/greet" function(req::HTTP.Request)
    return "hello world!"
end

# start the web server
serve()
```

## Request handlers

Request handlers are just functions, which means there are many valid ways to express them

- Request handlers don't have to be defined where the routes are. They can be imported from other modules and spread across multiple files

- Just like the request handlers, routes can be declared across multiple modules and files

```julia
using Oxygen

@get "/greet" function()
    "hello world!"
end

@get "/saluer" () -> begin
    "Bonjour le monde!"
end

@get "/saludar" () -> "¡Hola Mundo!"
@get "/salutare" f() = "ciao mondo!"

# This function can be declared in another module
function subtract(req, a::Float64, b::Float64)
  return a - b
end

# register foreign request handlers like this
@get "/subtract/{a}/{b}" subtract

# start the web server
serve()
```


## Path parameters

Path parameters are declared with braces and are passed directly to your request handler. 
```julia
using Oxygen

# use path params without type definitions (defaults to Strings)
@get "/add/{a}/{b}" function(req, a, b)
    return parse(Float64, a) + parse(Float64, b)
end

# use path params with type definitions (they are automatically converted)
@get "/multiply/{a}/{b}" function(req, a::Float64, b::Float64)
    return a * b
end

# The order of the parameters doesn't matter (just the name matters)
@get "/subtract/{a}/{b}" function(req, b::Int64, a::Int64)
    return a - b
end

# start the web server
serve()
```

## Query parameters

Use the `queryparams()` function to extract and parse parameters from the url

```julia
using Oxygen
using HTTP

@get "/query" function(req::HTTP.Request)
    # extract & return the query params from the request object
    return queryparams(req)
end

# start the web server
serve()
```

## Interpolating variables into endpoints

You can interpolate variables directly into the paths, which makes dynamically registering routes a breeze 

(Thanks to @anandijain for the idea)
```julia
using Oxygen

operations = Dict("add" => +, "multiply" => *)
for (pathname, operator) in operations
    @get "/$pathname/{a}/{b}" function (req, a::Float64, b::Float64)
        return operator(a, b)
    end
end

# start the web server
serve()
```

## Return JSON

All objects are automatically deserialized into JSON using the JSON3 library

```julia
using Oxygen
using HTTP

@get "/data" function(req::HTTP.Request)
    return Dict("message" => "hello!", "value" => 99.3)
end

# start the web server
serve()
```

## Deserialize & Serialize custom structs
Oxygen provides some out-of-the-box serialization & deserialization for most objects but requires the use of StructTypes when converting structs

```julia
using Oxygen
using HTTP
using StructTypes

struct Animal
    id::Int
    type::String
    name::String
end

# Add a supporting struct type definition so JSON3 can serialize & deserialize automatically
StructTypes.StructType(::Type{Animal}) = StructTypes.Struct()

@get "/get" function(req::HTTP.Request)
    # serialize struct into JSON automatically (because we used StructTypes)
    return Animal(1, "cat", "whiskers")
end

@post "/echo" function(req::HTTP.Request)
    # deserialize JSON from the request body into an Animal struct
    animal = json(req, Animal)
    # serialize struct back into JSON automatically (because we used StructTypes)
    return animal
end

# start the web server
serve()
```

## Routers

The `router()` function is an HOF (higher order function) that allows you to reuse the same path prefix & properties across multiple endpoints. This is helpful when your api starts to grow and you want to keep your path operations organized.

Below are the arguments the `router()` function can take:
```julia
router(prefix::String; tags::Vector, middleware::Vector, interval::Real)
```
- `tags` are used to organize endpoints in the autogenerated docs
- `middleware` is used to setup router & route-specific middleware
- `interval` is used to support repeat actions (*calling a request handler on a set interval in seconds*)

```julia
using Oxygen

# Any routes that use this router will be automatically grouped 
# under the 'math' tag in the autogenerated documenation
math = router("/math", tags=["math"])

# You can also assign route specific tags
@get math("/multiply/{a}/{b}", tags=["multiplication"]) function(req, a::Float64, b::Float64)
    return a * b
end

@get math("/divide/{a}/{b}") function(req, a::Float64, b::Float64)
    return a / b
end

serve()
```

## Repeat Tasks

The `router()` function has an `interval` parameter which is used to call
a request handler on a set interval (in seconds). 

**It's important to note that request handlers that use this property can't define additional function parameters outside of the default `HTTP.Request` parameter.**

In the example below, the `/repeat/hello` endpoint is called every 0.5 seconds and `"hello"` is printed to the console each time.

```julia
using Oxygen

repeat = router("/repeat", interval=0.5, tags=["repeat"])

@get repeat("/hello") function()
    println("hello")
end

# you can override properties by setting route specific values 
@get repeat("/bonjour", interval=1.5) function()
    println("bonjour")
end

serve()
```

## Mounting Static Files

You can mount static files using this handy macro which recursively searches a folder for files and mounts everything. All files are 
loaded into memory on startup.

```julia
using Oxygen

# mount all files inside the "content" folder under the "/static" path
@staticfiles("content", "static")

# start the web server
serve()
```

## Mounting Dynamic Files 

Similar to @staticfiles, this macro mounts each path and re-reads the file for each request. This means that any changes to the files after the server has started will be displayed.

```julia
using Oxygen

# mount all files inside the "content" folder under the "/dynamic" path
@dynamicfiles("content", "dynamic")

# start the web server
serve()
```
## Performance Tips

Disabling the internal logger can provide some massive performance gains, which can be helpful in some scenarios.
Anecdotally, i've seen a 2-3x speedup in `serve()` and a 4-5x speedup in `serveparallel()` performance.

```julia 
# This is how you disable internal logging in both modes
serve(access_log=nothing)
serveparallel(access_log=nothing)
```

## Logging

Oxygen provides a default logging format but allows you to customize the format using the `access_log` parameter. This functionality is available in both the `serve()` and `serveparallel()` functions.

You can read more about the logging options [here](https://juliaweb.github.io/HTTP.jl/stable/reference/#HTTP.@logfmt_str)

```julia 
# Uses the default logging format
serve()

# Customize the logging format 
serve(access_log=logfmt"[$time_iso8601] \"$request\" $status")

# Disable internal request logging 
serve(access_log=nothing)
```

## Middleware

Middleware functions make it easy to create custom workflows to intercept all incoming requests and outgoing responses.
They are executed in the same order they are passed in (from left to right).

They can be set at the application, router, and route layer with the `middleware` keyword argument. All middleware is additive and any middleware defined in these layers will be combined and executed.

Middleware will always be executed in the following order:

```
application -> router -> route
```

Now lets see some middleware in action:
```julia
using Oxygen
using HTTP

const CORS_HEADERS = [
    "Access-Control-Allow-Origin" => "*",
    "Access-Control-Allow-Headers" => "*",
    "Access-Control-Allow-Methods" => "POST, GET, OPTIONS"
]

# https://juliaweb.github.io/HTTP.jl/stable/examples/#Cors-Server
function CorsMiddleware(handler)
    return function(req::HTTP.Request)
        println("CORS middleware")
        # determine if this is a pre-flight request from the browser
        if HTTP.method(req)=="OPTIONS"
            return HTTP.Response(200, CORS_HEADERS)  
        else 
            return handler(req) # passes the request to the AuthMiddleware
        end
    end
end

function AuthMiddleware(handler)
    return function(req::HTTP.Request)
        println("Auth middleware")
        # ** NOT an actual security check ** #
        if !HTTP.headercontains(req, "Authorization", "true")
            return HTTP.Response(403)
        else 
            return handler(req) # passes the request to your application
        end
    end
end

function middleware1(handle)
    function(req)
        println("middleware1")
        handle(req)
    end
end

function middleware2(handle)
    function(req)
        println("middleware2")
        handle(req)
    end
end

# set middleware at the router level
math = router("math", middleware=[middleware1])

# set middleware at the route level 
@get math("/divide/{a}/{b}", middleware=[middleware2]) function(req, a::Float64, b::Float64)
    return a / b
end

# set application level middleware
serve(middleware=[CorsMiddleware, AuthMiddleware])
```

## Custom Response Serializers

If you don't want to use Oxygen's default response serializer, you can turn it off and add your own! Just create your own special middleware function to serialize the response and add it at the end of your own middleware chain. 

Both `serve()` and `serveparallel()` have a `serialize` keyword argument which can toggle off the default serializer.

```julia
using Oxygen
using HTTP
using JSON3

@get "/divide/{a}/{b}" function(req::HTTP.Request, a::Float64, b::Float64)
    return a / b
end

# This is just a regular middleware function
function myserializer(handle)
    function(req)
        try
          response = handle(req)
          # convert all responses to JSON
          return HTTP.Response(200, [], body=JSON3.write(response)) 
        catch error 
            @error "ERROR: " exception=(error, catch_backtrace())
            return HTTP.Response(500, "The Server encountered a problem")
        end 
    end
end

# make sure 'myserializer' is the last middleware function in this list
serve(middleware=[myserializer], serialize=false)
```

## Multithreading & Parallelism

For scenarios where you need to handle higher amounts of traffic, you can run Oxygen in a 
multithreaded mode. In order to utilize this mode, julia must have more than 1 thread to work with. You can start a julia session with 4 threads using the command below
```shell 
julia --threads 4
```

``serveparallel(queuesize=1024)`` Starts the webserver in streaming mode and spawns n - 1 worker 
threads. The ``queuesize`` parameter sets how many requests can be scheduled within the queue (a julia Channel)
before they start getting dropped. Each worker thread pops requests off the queue and handles them asynchronously within each thread. 

```julia
using Oxygen
using StructTypes
using Base.Threads

# Make the Atomic struct serializable
StructTypes.StructType(::Type{Atomic{Int64}}) = StructTypes.Struct()

x = Atomic{Int64}(0);

@get "/show" function()
    return x
end

@get "/increment" function()
    atomic_add!(x, 1)
    return x
end

# start the web server in parallel mode
serveparallel()
```

## Autogenerated Docs with Swagger

Swagger documentation is automatically generated for each route you register in your application. Only the route name, parameter types, and 200 & 500 responses are automatically created for you by default. 

You can view your generated documentation at `/docs`, and the schema
can be found under `/docs/schema`. Both of these values can be changed to whatever you want using the `configdocs()` function. You can also opt out of autogenerated docs entirely by calling the `disabledocs()` function 
before starting your application. 

To add additional details you can either use the built-in `mergeschema()` or `setschema()`
functions to directly modify the schema yourself or merge the generated schema from the `SwaggerMarkdown.jl` package (I'd recommend the latter)

Below is an example of how to merge the schema generated from the `SwaggerMarkdown.jl` package.
```julia
using Oxygen
using SwaggerMarkdown

# Here's an example of how you can merge autogenerated docs from SwaggerMarkdown.jl into your api
@swagger """
/divide/{a}/{b}:
  get:
    description: Return the result of a / b
    parameters:
      - name: a
        in: path
        required: true
        description: this is the value of the numerator 
        schema:
          type : number
    responses:
      '200':
        description: Successfully returned an number.
"""
@get "/divide/{a}/{b}" function (req, a::Float64, b::Float64)
    return a / b
end

# title and version are required
info = Dict("title" => "My Demo Api", "version" => "1.0.0")
openApi = OpenAPI("3.0", info)
swagger_document = build(openApi)
  
# merge the SwaggerMarkdown schema with the internal schema
mergeschema(swagger_document)

# start the web server
serve()
```

Below is an example of how to manually modify the schema
```julia
using Oxygen
using SwaggerMarkdown

# Only the basic information is parsed from this route when generating docs
@get "/multiply/{a}/{b}" function (req, a::Float64, b::Float64)
    return a * b
end

# Here's an example of how to update a part of the schema yourself
mergeschema("/multiply/{a}/{b}", 
  Dict(
    "get" => Dict(
      "description" => "return the result of a * b"
    )
  )
)

# Here's another example of how to update a part of the schema yourself, but this way allows you to modify other properties defined at the root of the schema (title, summary, etc.)
mergeschema(
  Dict(
    "paths" => Dict(
      "/multiply/{a}/{b}" => Dict(
        "get" => Dict(
          "description" => "return the result of a * b"
        )
      )
    )
  )
)
```

# Common Issues & Tips

## Problems working with Julia's REPL

This is a recurring issue that occurs when writing and testing code in the REPL. Often, people find that their changes are not reflected when they rerun the server. The reason for this is that all the routing utilities are defined as macros, and they are only executed during the precompilation stage. To have your changes take effect, you need to move your route declarations to the `__init__()` function in your module.

```julia
module OxygenExample
using Oxygen
using HTTP

# is called whenever you load this module
function __init__()
    @get "/greet" function(req::HTTP.Request)
        return "hello world!"
    end
end

# you can call this function from the REPL to start the server
function runserver()
    serve()
end

end 
```



# API Reference (macros)

#### @get, @post, @put, @patch, @delete
```julia
  @get(path, func)
```
| Parameter | Type     | Description                |
| :-------- | :------- | :------------------------- |
| `path` | `string` or `router()` | **Required**. The route to register |
| `func` | `function` | **Required**. The request handler for this route |

Used to register a function to a specific endpoint to handle that corresponding type of request

#### @route
```julia
  @route(methods, path, func)
```
| Parameter | Type     | Description                |
| :-------- | :------- | :------------------------- |
| `methods` | `array` | **Required**. The types of HTTP requests to register to this route|
| `path` | `string` or `router()` | **Required**. The route to register |
| `func` | `function` | **Required**. The request handler for this route |

Low-level macro that allows a route to be handle multiple request types


#### @staticfiles
```julia
  @staticfiles(folder, mount)
```
| Parameter | Type     | Description                |
| :-------- | :------- | :------------------------- |
| `folder` | `string` | **Required**. The folder to serve files from |
| `mountdir` | `string` | The root endpoint to mount files under (default is "static")|

Serve all static files within a folder. This function recursively searches a directory
and mounts all files under the mount directory using their relative paths.


### Request helper functions

#### html()
```julia
  html(content, status, headers)
```
| Parameter | Type     | Description                |
| :-------- | :------- | :------------------------- |
| `content` | `string` | **Required**. The string to be returned as HTML |
| `status` | `integer` | The HTTP response code (default is 200)|
| `headers` | `dict` | The headers for the HTTP response (default has content-type header set to "text/html; charset=utf-8") |

Helper function to designate when content should be returned as HTML


#### queryparams()
```julia
  queryparams(request)
```
| Parameter | Type     | Description                |
| :-------- | :------- | :------------------------- |
| `req` | `HTTP.Request` | **Required**. The HTTP request object |

Returns the query parameters from a request as a Dict()

### Body Functions

#### text()
```julia
  text(request)
```
| Parameter | Type     | Description                |
| :-------- | :------- | :------------------------- |
| `req` | `HTTP.Request` | **Required**. The HTTP request object |

Returns the body of a request as a string

#### binary()
```julia
  binary(request)
```
| Parameter | Type     | Description                |
| :-------- | :------- | :------------------------- |
| `req` | `HTTP.Request` | **Required**. The HTTP request object |

Returns the body of a request as a binary file (returns a vector of `UInt8`s)

#### json()
```julia
  json(request, classtype)
```
| Parameter | Type     | Description                |
| :-------- | :------- | :------------------------- |
| `req` | `HTTP.Request` | **Required**. The HTTP request object |
| `classtype` | `struct` | A struct to deserialize a JSON object into |

Deserialize the body of a request into a julia struct 
