---
title: "ControlOverValidationException"
date: 2023-04-05
draft: false
description: "How to deal with Validation Exception on Laravel"
tags: ["Programming", "PHP", "Laravel"]
showAuthor: false
sharingLinks: false
showTaxonomies: true
---

## Introduction

Laravel is a well known and very complete framework. With over 29k starts on GitHub it has a solid and very opinionated vison about how web systems should be developed since defining things like Routing, Validation and Error handling as well as more complex subjects like Queues, Websockets and Caching. Besides the examples I mentioned - and there is even more if you take a look at their [docs](https://laravel.com/docs/10.x) - they also have a rich ecosystem with a lot of tools to help developers with their job.

Today I'll be focusing on two parts of the framework in this we will explore how error handling can be done together with Laravel's validation system.

## Custom Exceptions

Digressing a little, one powerful thing we can do as developers is to use CustomExceptions on our code. Why? because exceptions unlocks the ability of having control over the flow of execution precisely by scoping the code into one or many blocks. First block is set by the reserved word try, and, in this block is where the code we want to execute is put. Besides the try block, we also have as many catch blocks we need, and in those, we scope code execution through specifics exceptions that are thrown. In other words, we start our execution path from the try block and inside of it we can throw exceptions that can be catched by the catch blocks. In addition, exceptions add a lot of readability  to the code base due to its simple and clear struture.

for instance, below an example - using pseudocode -  hipotetical rest api endpoint that returns the user data by id.

```php
class UserService {
	public function findUserById(id) {
		$user = User::find(id)
		if (!user)
			throw new UserNotFoundException();
	  
		return user;
	}
}


UserController extends Controller {
	// ...other methods above

	public function show() {
		try {
			$user = $userService->findUserById(1);
			return response()->json(['data' => user], 200);
		} catch(UserNotFoundException $e) {
			return response->json([], 404);
		}
	}
}

```

Isn't that pretty clean? first we try to execute the findByUserId and if that works we just return what we get from that, on the other hand, if the user with that id does not exist then we throw and exception that will be catched by the catch block and from there we can handle it by sending a proper response as we need.

Also, a try catch block can have as many catch blocks we would need and its syntax is always defined same way with the word catch followed by parentesis and the type of exception that is will be catch.

for instance, let's enhance the example above and assume that for some reason in this system users have privacy levels and depending on the privacy level I won't allow that user data to be returned.

```php
class UserService {
	public function findUserById(id) {
		$user = User::find(id)
		if (!user)
			throw new UserNotFoundException();
	  
		return user;
	}

	public function checkPrivacyLevel(user) {
		if (user.getPravacyLevel() === 'HIGH')
			throw new UserWithHighPrivacyLevelException()
	}
}

UserController extends Controller {
	// ...other methods above

	public function show()
	{
		try {
			$user = $userService->findUserById(1);
			$userService->checkUserPrivacyLevel(user);
			
			return response()->json(['data' => user], 200);
		} catch(UserNotFoundException $e) {
			return response->json([], 404);
		} catch(UserWithHighPrivacyLevelException$e) {
			return response->json([], 403);
		}
	}
}

```

Nice, eh? Now we know a little about custom exceptions let's see how Laravel's allows us to use it.

## Exception Handler

Laravel ships with an exception handler to help people organize their custom exceptions. I won't focus on all the properties and methods of the handler and neither will go deep in its inner guts to show how it works, specifically, I will focus on one specific method that we will use to handle exceptions for Laravel validation system.

Without further ado, let's create a class called BadRequestException like the following.


```php
use Exception;
use Illuminate\Http\JsonResponse;

class BadRequestException extends Exception {
	/**
	 *
	 * Returns 400 when a FormRequest fails
	 * any validation
	 *
	 */
	public function render(): JsonResponse
	{
		 return response()->json([], 400);
	}
}
```

By doing this Laravel's exception handler will turn your exception into a JsonResponse by calling the render method. Now that we have our exception configured let's take a look at how we can hook that up with Laravel's FormRequest.

According to Laravel, FormRequests

> ... are custom request classes that encapsulate their own validation and authorization logic.

In those classes, we have to implement a method called rules that will define the validation of your request object, something like the following:


```php

/**
 * Get the validation rules that apply to the request.
 *
 * @return array<string, \Illuminate\Contracts\Validation\Rule|array|string>
 */
public function rules(): array
{
	return [
		'title' => 'required|unique:posts|max:255',
		'body' => 'required',
	];
}
```

We can read about all that fun stuff in their docs. Right now let's do something more fun. Let's open [this](https://github.com/laravel/framework/blob/be2ddb5c31b0b9ebc2738d9f37a9d4c960aa3199/src/Illuminate/Foundation/Http/FormRequest.php) file and scroll till line 146... there we can see what happens when any rule fails on the form request and... HA a ValidatioException is thrown!

Now how can we have control over that? Well in order to take control over it we can override this method in our FormRequest class, something like this.

```php

class UserFormRequest extends FormRequest
{
	// ... other methods above

	/**
	 * Get the validation rules that apply to the request.
	 *
	 * @return array<string, \Illuminate\Contracts\Validation\Rule|array|string>
	 */
	public function rules(): array
	{
		return [
			'title' => 'required|unique:posts|max:255',
			'body' => 'required',
		];
	}

	/**  
	 * Handle a failed validation attempt. *
	 * @param  \Illuminate\Contracts\Validation\Validator  $validator  
	 * @return void  
	 *
	 * @throws BadRequestException 
	 */
	protected function failedValidation(Validator $validator): void
	{
		throw new BadRequestException($validator);
	}
}
```


Now the only thing we need to do is to tweak the BadRequestException class to make it receive the $validator as a parameter like below.


```php
use Exception;
use Illuminate\Http\JsonResponse;

class BadRequestException extends Exception {
	private $validator;

	public function __construct(Validator $validator)
	{
		$this->validator = $validator;
	}

	/**
	 *
	 * Returns 400 when a FormRequest fails
	 * any validation
	 *
	 */
	public function render(): JsonResponse
	{
		 return response()->json([], 400);
	}
}
```

You could even use the $validator to send something back from your response or even build up a trait that does that! And now you have full control over your request validation :-).

## References

https://laravel.com/docs/10.x/validation#form-request-validation
https://laravel.com/docs/10.x/errors#renderable-exceptions
https://github.com/laravel/framework/blob/be2ddb5c31b0b9ebc2738d9f37a9d4c960aa3199/src/Illuminate/Foundation/Http/FormRequest.php