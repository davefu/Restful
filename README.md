Simple Nette REST API
=====================
This repository is being developed. Project is for study purposes. Do not use it on production.

### Content
- [Requirements](#requirements)
- [Installation & setup](#installation--setup)
- [Neon configuration](#neon-configuration)
- [Sample usage](#sample-usage)
- [Simple CRUD resources](#simple-crud-resources)
- [Accessing input data](#accessing-input-data)
- [Security & authentication](#security--authentication)
- [JSONP support](#jsonp-support)

Requirements
------------
Drahak/Restful requires PHP version 5.3.0 or higher. The only production dependency is [Nette framework 2.0.x](http://www.nette.org).

Installation & setup
--------------------
The easist way is to use [Composer](http://doc.nette.org/en/composer)

	$ composer require drahak/restful:@dev

Then add following code to your app bootstrap file before creating container:

```php
Drahak\Restful\DI\Extension::install($configurator);
```

Neon configuration
------------------
You can configure Drahak\Restful library in config.neon in section `restful`:

```yaml
restful:
	cacheDir: '%tempDir%/cache'
	jsonpKey: 'jsonp'
	routes:
		prefix: resources
		module: 'RestApi'
		autoGenerated: TRUE
		panel: TRUE
	security:
		privateKey: 'my-secret-api-key'
		requestTimeKey: 'timestamp'
		requestTimeout: 300
```

- `cacheDir`: not much to say, just directory where to store cache
- `jsonpKey`: sets query parameter name, which enables [JSONP envelope mode](#jsonp-support). Set this to FALSE if you wish to disable it.
- `routes.prefix`: mask prefix to resource routes (**only for auto generated routes**)
- `routes.module`: default module to resource routes (**only for auto generated routes**)
- `routes.autoGenerated`: if `TRUE` the library auto generate resource routes from Presenter action method annotations (see below)
- `routes.panel`: if `TRUE` the resource routes panel will appear in your nette debug bar
- `security.privateKey`: private key to hash secured requests
- `security.requestTimeKey`: key in request body, where to find request timestamp (see below - [Security & authentication](#security--authentication))
- `security.requestTimeout`: maximal request timestamp age

#### Resource routes panel
It is enabled by default but you can disable it by setting `restful.routes.panel` to `FALSE`. This panel show you all REST API resources routes (exactly all routes in default route list which implements `IResourceRouter` interface). This is useful e.g. for developers who develop client application, so they have all API resource routes in one place.
![REST API resource routes panel](http://files.drahak.eu/restful-routes-panel.png "REST API resource routes panel")

Sample usage
------------

Create `BasePresenter`:

```php
<?php
namespace ResourcesModule;

use Drahak\Restful\Application\ResourcePresenter;
use Drahak\Restful\IResource;

/**
 * BasePresenter
 * @package ResourcesModule
 * @author Drahomír Hanák
 */
abstract class BasePresenter extends ResourcePresenter
{

    /** @var string */
    protected $defaultContentType = IResource::JSON;

}
```

The `defaultContentType` property determines how to generate response from your resource. Can be overridden by request `Accept` header. Library checks the header for `application/xml`, `application/json`, `application/x-data-url` and `text/x-query` and keep an order in `Accept` header.

Note: If you call `$presenter->sendResource()` method with a mime type in first parameter, API will accept only this one.

```php
<?php
namespace ResourcesModule;

use Drahak\Restful\IResource;

/**
 * SamplePresenter resource
 * @package ResourcesModule
 * @author Drahomír Hanák
 */
class SamplePresenter extends BasePresenter
{

   protected $typeMap = array(
       'json' => IResource::JSON,
       'xml' => IResource::XML
   );

   /**
    * @GET sample[.<type xml|json>]
    */
   public function actionContent($type = 'json')
   {
       $this->resource->title = 'REST API';
       $this->resource->subtitle = '';
       $this->sendResource($this->typeMap[$type]);
   }

   /**
    * @GET sample/detail
    */
   public function actionDetail()
   {
       $this->resource->message = 'Hello world';
   }

}
```

See `@GET` annotation. There are also available annotations `@POST`, `@PUT`, `@HEAD`, `@DELETE`. This allows Drahak\Restful library to generate API routes for you so you don't need to do it manualy. But it's not neccessary! You can define your routes using `IResourceRoute` or its default implementation such as:
```php
<?php
use Drahak\Restful\Application\Routes\ResourceRoute;

$anyRouteList[] = new ResourceRoute('sample[.<type xml|json>]', 'Resources:Sample:content', ResourceRoute::GET);
```

There is only one more parameter unlike the Nette default Route, the request method. This allows you to generate same URL for e.g. GET and POST method. You can pass this parameter to route as a flag so you can combine more request methods such as `ResourceRoute::GET | ResourceRoute::POST` to listen on GET and POST request method in the same route.

You can also define action names dictionary for each reqest method:

```php
<?php
new ResourceRoute('myResourceName', array(
    'presenter' => 'MyResourcePresenter',
    'action' => array(
        ResourceRoute::GET => 'content',
        ResourceRoute::DELETE => 'delete'
    )
), ResourceRoute::GET | ResourceRoute::DELETE);
```

Simple CRUD resources
---------------------
Well it's nice but in many cases I define only CRUD operations so how can I do it more intuitively? Use `CrudRoute`! This child of `ResourceRoute` predefines base CRUD operations for you. Namely, it is `Presenter:create` for POST method, `Presenter:read` for GET, `Presenter:update` for PUT and `Presenter:delete` for DELETE. Then your router will look like this:

```php
<?php
new CrudRoute('<module>/crud', 'MyResourcePresenter');
```
Note the second parameter, metadata. You can define only Presenter not action name. This is because the action name will be replaced by value from actionDictionary (`[CrudRoute::POST => 'create', CrudRoute::GET => 'read', CrudRoute::PUT => 'update', CrudRoute::DELETE => 'delete']`) which is property of `ResourceRoute` so even of `CrudRoute` since it is its child. Also note that we don't have to set flags. Default flags are setted to `CrudRoute::CRUD` so the route will match all request methods.

Then you can simple define your CRUD resource presenter:

```php
<?php
namespace ResourcesModule;

/**
 * CRUD resource presenter
 * @package ResourcesModule
 * @author Drahomír Hanák
 */
class CrudPresenter extends BasePresenter
{

    public function actionCreate()
    {
        $this->resource->action = 'Create';
    }

    public function actionRead()
    {
        $this->resource->action = 'Read';
    }

    public function actionUpdate()
    {
        $this->resource->action = 'Update';
    }

    public function actionDelete()
    {
        $this->resource->action = 'Delete';
    }

}
```

Note: every request method can be overridden if you specify `X-HTTP-Method-Override` header in request or by adding query parameter `__method` to URL.

Accessing input data
--------------------
If you want to build REST API, you may also want to access query input data for all request methods (GET, POST, PUT, DELETE and HEAD). So the library defines input parser, which reads data and parse it to an array. Data are fetched from query string or from request body and parsed by `IMapper`. First the library looks for request body. If it's not empty it checks `Content-Type` header and determines correct mapper (e.g. for `application/json` -> `JsonMapper` etc.) Then, if request body is empty, try to get POST data and at the end even URL query data.

```php
<?php
namespace ResourcesModule;

/**
 * Sample resource
 * @package ResourcesModule
 * @author Drahomír Hanák
 */
class SamplePresenter extends BasePresenter
{

	/**
	 * @PUT <module>/sample
	 */
	public function actionUpdate()
	{
		$this->resource->message = isset($this->input->message) ? $this->input->message : 'no message';
	}

}
```
Good thing about it is that you don't care of request method. Nette Drahak REST API library will choose correct Input parser for you but it's still up to you, how to handle it. There is available `InputIterator` so you can iterate through input in presenter or use it in your own input parser as iterator.

Security & authentication
-------------------------
The library provides base support of secure API calls. It's based on sending hashed data with private key. Authentication process is as follows:

### Understanding authentication process
- Client: append request timestamp to request body.
- Client: hash all data with `hash_hmac` (sha256 algorithm) and with private key. Then append generated hash to request as `X-HTTP-AUTH-TOKEN` header (by default).
- Client: sends request to server.
- Server: accepts client's request and calculate hash in the same way as client (using abstract template class `AuthenticationProcess`)
- Server: compares client's hash with hash that it generated in previous step.
- Server: also checks request timestamp and make difference. If it's bigger then 300 (5 minutes) throws exception. (this avoid something called [Replay Attack](http://en.wikipedia.org/wiki/Replay_attack))
- Server: catches any `SecurityException` that throws `AuthenticationProcess` and provides error response.

Default `AuthenticationProcess` is `NullAuthentication` so all requests are unsecured. You can use `SecuredAuthentication` to secure your resources. To do so, just set this authentication process to `AuthenticationContext` in `restful.authentication` or `$presenter->authentication`.

```php
<?php
namespace ResourcesModule;

use Drahak\Restful\Security\SecuredAuthentication;

/**
 * CRUD resource presenter
 * @package ResourcesModule
 * @author Drahomír Hanák
 */
class CrudPresenter extends BasePresenter
{

	/** @var SecuredAuthentication */
	private $securedAuthentication;

	/**
	 * Inject secured authentication process
	 * @param SecuredAuthentication $auth
	 */
	public function injectSecuredAuthentication(SecuredAuthentication $auth)
	{
		$this->securedAuthentication = $auth;
	}

	protected function startup()
	{
		parent::startup();
		$this->authentication->setAuthProcess($this->securedAuthentication);
	}

	// your secured resource action
}
```

##### Never send private key!

JSONP support
-------------
If you want to access your API resources by JavaScript on remote host, you can't make normal AJAX request on API. So JSONP is alternative how to do it. In JSONP request you load your API resource as a JavaScript using standard <script> tag in HTML. API wraps JSON string to a callback function parameter. It's actually pretty simple but it needs special care. For example you can't access response headers or status code. You can wrap these headers and status code to all your resources but this is not good for normal API clients, which can access header information. The library allows you to add special query parameter `jsonp` (name depends on your configuration, this is default value). If you access resource with `?jsonp=callback` API automatically determines JSONP mode and wraps all resources to following JavaScript:

```javascript
callback({
	"response": {
		"yourResourceData": "here"
	},
	"status_code": 200,
	"headers": {
		"X-Powered-By": "Nette framework",
		...
	}
})
```
**Note** : the function name. This is name from `jsonp` query parameter. This string is "webalized" by `Nette\Utils\Strings::webalize(jsonp, NULL, FALSE)`. If you set `jsonpKey` to `FALSE` or `NULL` in configuration, you totally disable JSONP mode for all your API resources. Then you can trigger it manually. Just set `IResource` `$contentType` property to `IResource::JSONP`.

**Also note** : if this option is enabled and client adds `jsonp` parameter to query string, no matter what you set to `$presenter->resource->contentType` it will produce `JsonpResponse`.

___

TODO list
---------
What I plan to (sometime) implement.

- Better API resources panel. There are to many records in the table.
- `@resource` annotation (presenter class) to auto generate CrudRoute
- Refactor `RouteListFactory` for easier scalability

So that's it. Enjoy and hope you like it!