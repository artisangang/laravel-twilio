laravel-twilio
================

A Twilio port for Laravel (4.2+)


## Install


Add the package to your `composer.json` and run composer update:

	{
	  "require": {
	  	...
	    "j42/laravel-twilio": "dev-master"
	  }
	}

Then add the service providers and facades to `config/app.php`

```php
	'J42\LaravelTwilio\LaravelTwilioServiceProvider',
```
...
```php
	'Twilio'		  => 'J42\LaravelTwilio\LaravelTwilioFacade'
```


## Configure

Generate the package config files by running `php artisan config:publish j42/laravel-twilio`, and adjust the relevant fields:

```php
	return [

		'key'	=> 'YOURAPIKEY',						// Public key
		'token'	=> 'YOURSECRETTOKEN',					// Private key
		'from'	=> '9999999999',						// Default From Address 
		'twiml'	=> 'https://yourdomain.com/base_path',	// TWIML Hosted Base Path

		// Default Features When Requesting Numbers (override via `numbersNear` method)
		'features'	=> [
	    	'SmsEnabled'	=> true,
	    	'VoiceEnabled'	=> false,
	    	'MmsEnabled'	=> false,
	    ]

	];
```

## Phone Verification

Since phone verification is such a common use case, I created a simple flow to automate this.

This package automatically installs the following routes (`GET` or `POST` allowed):

	/twilio/verify

### Request a Token

Initiate an HTTP `GET` or `POST` request to `/twilio/verify` with the following parameters:

- `phone` phone number (parsed as string)
- `method` ('sms' or 'call')

The numeric token is set in a cookie and has a 2 minute TTL during which it is valid.

**Returns:**

```php
	// Get token (method can be either 'sms' or 'call')
	file_get_contents('<yourdomain>/twilio/verify?phone=0000000000&method=sms');
	
	/* 
		{
			status: 'success',
			data: {
				phone: '0000000000',	// User's phone
				status: 'queued'		// Twilio response status
			}
		}
	*/
```

### Verify a Token

Initiate an HTTP `GET` or `POST` re	uest to `/twilio/verify/` with the following parameters:

- `code` numeric code entered by user

If properly verified, the full object will be returned:

```php
	// Verify token
	file_get_contents('/twilio/verify?code=00000');
	
	/*
		{
			status: 'success',
			data: {
				code: '00000',			// Initial Generated Code
				phone: '0000000000',	// User's phone
				valid: true
			}
		}
	*/
```


### Post-Verification (Success)

Once the code has been confirmed, the verified data is available via `Cookie` with a 5 minute TTL.  An HTTP request to `/twilio/verify` (with or without any parameters) will return:

```php
	file_get_contents('/twilio/verify');
	
	/*
		{
			status: 'success',
			data: {
				code: '00000',			// Initial Generated Code
				phone: '0000000000',	// User's phone
				valid: true
			}
		}
	*/
```


#### Advanced Usage

Sometimes you may need to handle additional logic in a controller of your own.  By including a handy interface, this becomes easy:

**Define the route overrides (whichever suits your preference, or, both)**

```php
	\Route::any('twilio/verify', [
		'uses'	=> 'YourController@verify'
	]);

	\Route::any('api/twilio/verify', [
		'uses'	=> 'YourController@verify'
	]);
```

**Create your controller, extending `J42\LaravelTwilio\TwilioVerify`**

```php
	use J42\LaravelTwilio\TwilioVerify;

	class TwilioController extends TwilioVerify {

		// Verify Phone
		public function verify() {
			
			// Your pre-verification logic

			// Magic
			$response = parent::verify();

			// Your post-verification logic
			// Cookie::get('twilio::phone') now available === json_decode($response)['data']

			return $response;

		}

	}
```

**Define your functionality as needed, making sure to call `parent::verify();` to handle the default events.  If you need to access the cookie directly you may do so via: `Cookie::get('twilio::phone')`.**


## SMS

How to interact with Twilio's REST-based SMS methods.

####Send an SMS

```php
Twilio::sms([
	// From (optional -- if unsupplied, will be taken from default Config::get('twilio::config.from'))
	'from'		=> '<your twilio #>'
	// Array of recipients
	'to'		=> ['19999999999'],
	// Text Message
	'message'	=> 'Contents of the text message go here'
]);
```


## Call

How to interact with Twilio's REST-based call initiation methods.

####Initiate a Call (TWIML Endpoint)

```php
Twilio::call([
	// From (optional -- if unsupplied, will be taken from default Config::get('twilio::config.from'))
	'from'		=> '<your twilio #>'
	// Array of recipients
	'to'		=> ['19999999999'],
	// Relative path to twiml document/endpoint (combined with Config::get('twilio::config.twiml') to form an absolute URL endpoint)
	'twiml'		=> 'twilio/verify/twiml'
]);

// Response Statuses:
// QUEUED, RINGING, IN-PROGRESS, COMPLETED, FAILED, BUSY or NO_ANSWER.
```


## Request Local Numbers

You can also request local numbers (to be used in the 'from' field) via any of the attributes available in the SDK client.  **Currently US only.  If you want to adapt this (feel free to fork it) you may do so easily by abstracting the geoList parameters to the configuration file.**


```php

// Near Area Code (With MMS Capability)
Twilio::numbersNear([ 'AreaCode' => '415' ], ['MmsEnabled' => true]);	// Second parameter is an optional array of features (SmsEnabled, VoiceEnabled, MmsEnabled)

// Near Another Phone #
Twilio::numbersNear([ 
	'NearNumber' => '415XXXXXX',	// Other Number
	'Distance'	 => '50'			// Miles (optional, default: 25)
]);

// Near A City (any combination allowed)
Twilio::numbersNear([ 
	'InRegion'		=> 'CA',				// State/Region/Province Code
	'InPostalCode'	=> '90017'				// Postal code?
]);

// Near Lat/Long Coordinates
Twilio::numbersNear([
	'NearLatLong'	=> '37.840699,-122.461853',
	'Distance'		=> '50'
]);

// ... you get the idea.  Most fields can be mixed and matched arbitrarily, but if you are wondering, test it out for yourself!

```

##### By Regex

A pattern to match phone numbers on. Valid characters are '*' and [0-9a-zA-Z]. The '*' character will match any single digit. See Example 2 and Example 3 below.

```php

// By Regex 
// Valid characters are '*' and [0-9a-zA-Z]. The '*' character will match any single digit. 

Twilio::numbersNear([ 'Contains' => 'J42' ]);			// Matches String
Twilio::numbersNear([ 'Contains' => '510555****' ]);	// Matches Pattern

```