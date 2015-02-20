JSON API helpers for Laravel 5
=====

Make it a breeze to create a [jsonapi.org](http://jsonapi.org/) compliant API with Laravel 5.

Installation
-----

Add `echo-it/laravel-jsonapi` to your composer.json dependency list and run `composer update`.

### Requirements

* PHP 5.4+
* Laravel 5


Using laravel-jsonapi
-----

This library is made with the concept of exposing models in mind, as found in the RESTful API approach.

In few steps you can expose your models:

1. **Create a route to direct the requests**

    In this example, we use a generic route for all models and HTTP methods:

    ```php
    Route::any('{model}/{id?}', 'ApiController@handleRequest');
    ```

2. **Create your controller to handle the request**

    Your controller is responsible to handling input, instantiating a handler class and returning the response.

    ```php
    <?php namespace App\Http\Controllers;

use EchoIt\JsonApi\Request as ApiRequest;
use EchoIt\JsonApi\ErrorResponse as ApiErrorResponse;
use EchoIt\JsonApi\Exception as ApiException;
use Request;

class ApiController extends Controller
{
    public function handleRequest($modelName, $id = null)
    {
        /**
         * Create handler name from model name
         * @var string
         */
        $handlerClass = 'App\\Handlers\\' . ucfirst($modelName) . 'Handler';

        if (class_exists($handlerClass)) {
			$url = Request::url();
            $method = Request::method();
            $include = ($i = Request::input('include')) ? explode(',', $i) : $i;
			$sort = ($i = Request::input('sort')) ? explode(',', $i) : $i;
			$filter = ($i = Request::except('sort', 'include')) ? $i : [];
			$content = Request::getContent();

            $request = new ApiRequest($url, $method, $id, $content, $include, $sort, $filter);
            $handler = new $handlerClass($request);

            // A handler can throw EchoIt\JsonApi\Exception which must be gracefully handled to give proper response
            try {
                $res = $handler->fulfillRequest();
            } catch (ApiException $e) {
                return $e->response();
            }
			
            return $res->toJsonResponse();
        }

        // If a handler class does not exist for requested model, it is not considered to be exposed in the API
        return new ApiErrorResponse(404, 404, 'Entity not found');
    }
}
    ```

3. **Create a handler for your model**

    A handler is responsible for exposing a single model.

    In this example we have create a handler which supports the following requests:

    * GET /users (ie. handleGet function)
    * GET /users/[id] (ie. handleGet function)
    * PUT /users/[id] (ie. handlePut function)
    
    Requests are automatically routed to appropriate handle functions.

    ```php
<?php namespace App\Handlers;

use Symfony\Component\HttpFoundation\Response;
use App\Models\User;

use EchoIt\JsonApi\Exception as ApiException;
use EchoIt\JsonApi\Request as ApiRequest;
use EchoIt\JsonApi\Handler as ApiHandler;
use Request;

/**
 * Handles API requests for Users.
 */
class UsersHandler extends ApiHandler
{
	const ERROR_SCOPE = 1024;
	
	/*
	* List of relations that can be included in response.
	* (eg. 'friend' could be included with ?include=friend)
	*/
	protected static $exposedRelations = [];
	
	/**
	 * Handles GET requests. 
	 * @param EchoIt\JsonApi\Request $request
	 * @return EchoIt\JsonApi\Model
	 */
	public function handleGet(ApiRequest $request)
	{
		//use default GET functionality, which includes
		//handling of sorting and filtering. 
		return $this->handleGetDefault($request, new User);
	}
	
	/**
	 * Handles PUT requests. 
	 * @param EchoIt\JsonApi\Request $request
	 * @return EchoIt\JsonApi\Model
	 */
	public function handlePut(ApiRequest $request)
	{
		//use default PUT functionality
		return $this->handlePutDefault($request, new User);
	}
}
    ```

    > **Note:** Extend your model from `EchoIt\JsonApi\Model` rather than `Eloquent` to get the proper response for linked resources.


Current features
-----

According to [jsonapi.org](http://jsonapi.org):

* [Resource Representations](http://jsonapi.org/format/#document-structure-resource-representations) as resource objects
* [Resource Relationships](http://jsonapi.org/format/#document-structure-resource-relationships)
   * Only through [Inclusion of Linked Resources](http://jsonapi.org/format/#fetching-includes)
* [Compound Documents](http://jsonapi.org/format/#document-structure-compound-documents)
* [Sorting](http://jsonapi.org/format/#fetching-sorting)
* [Filtering](http://jsonapi.org/format/#fetching-filtering)

Wishlist
-----

* Nested requests to fetch relations, e.g. /users/[id]/friends
* [Resource URLs](http://jsonapi.org/format/#document-structure-resource-urls)
* Requests for multiple [individual resources](http://jsonapi.org/format/#urls-individual-resources), e.g. `/users/1,2,3`
* [Sparse Fieldsets](http://jsonapi.org/format/#fetching-sparse-fieldsets)

* Some kind of caching mechanism
