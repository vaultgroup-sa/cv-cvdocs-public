# Authorization

A few API endpoints allow anonymous access.
All the rest require authorization using `Bearer` token which is JWT:
`Authorization: Bearer <JWT>`

For details about authorized client types (`unit`, `user`, `supervisor`) and permissions see [security] document.

# Actuator

Every application (microservice) exposes two anonymous (no authorization required) actuator endpoints:

* `/actuator/health` – a simple health check endpoint;
* `/actuator/info` – exposes some information about application (like artifact, version and timestamp of build).

# General

All the date/time values across all API endpoints are represented in ISO-8601 timestamp format unless explicitly stated
otherwise.
The part which represents fractions of second is optional (both examples are valid):

* `2022-05-05T10:19:03.432Z`
* `2022-05-05T10:19:03Z`

In case of error a proper HTTP response code is used and the `application/json` response body format is as follows:

```json
[
  {
    "message": "Certain error occurred"
  },
  {
    "field": "unit_id",
    "message": "Invalid UUID value"
  },
  ...
]
```

There is an array of one or more error objects. Each one must have `message` property which contains human-readable
description or what went wrong.
If it's an error that is related to a certain request param, path variable or form field then the error object will also
have `field` property.

There are plenty of validation errors caused by malformed or unexpected values of certain request parameters
or `application/json` request body object properties. So keep that in mind and make sure to provide generic error
handling.

# Audit endpoints

## POST `/audit/log`

Submit unit audit log(s) for processing/storing.

#### Access: `unit`, no permissions required

#### Request body: `application/json`

```json
[
  {
    "code": 100,
    "app": "cv",
    "priority": "low",
    "facility": "cvmain",
    "capture_time": "2022-05-05T10:18:41Z",
    "description": "Test log #1",
    "level": "Debug"
  },
  {
    "code": 101,
    "app": "cv",
    "priority": "norm",
    "facility": "cvmain",
    "capture_time": "2022-05-05T10:18:53Z",
    "description": "Test log #2",
    "level": "Debug"
  },
  {
    "code": 102,
    "app": "cv",
    "priority": "med",
    "facility": "cvmain",
    "capture_time": "2022-05-05T10:19:03Z",
    "description": "Test log #3",
    "level": "Debug"
  },
  {
    "code": 103,
    "app": "cv",
    "priority": "high",
    "facility": "cvmain",
    "capture_time": "2022-05-05T10:19:19Z",
    "description": "Test log #4",
    "level": "Debug",
    "parameters": "{\"foo\": \"bar\"}"
  }
]
```

*Required fields:*

* `code` an integer number that simply indicates the type of log entry;
* `capture_time` the timestamp of when the log entry was created;
* `description` a human-readable log message.

*Optional fields:*

* `app` an application name (either "cvmain" or any client-specific app name);
* `priority` an indication of how important this message is (valid values are: "vvv_low", "vv_low", "v_low", "low", "norm", "med", "high", "v_high", "vv_high", "vvv_high", "extreme");
* `facility` an additional information about the source of message (e.g. application module name);
* `level` a logging level which indicates importance of log entry (severity of an issue if there's one);
* `parameters` an escaped json object (a string) which may be specified to describe log entry in formalized manner.

#### Response

*Success:*

All the submitted log entries are processed (invalid entries are ignored) and should not be submitted again.

* Response code: `200 OK`
* Response body: `none`

*Validation failure:*

Request body is missing or invalid.

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Missing or invalid request body"
    }
  ]
}
```

## GET `/audit/dashboard/events`

Get all dashboard events.

#### Access: `user` or `supervisor` having `unit.logs.view` permission

#### Request params:

*Optional:*

* `account_id` filter by unit's account (only available to `supervisor`);
* `site_id` filter by unit's site;
* `unit_id` filter by unit;
* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `25`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `type`, `title`, `message`, `account_id`, `account_name`,
  `site_id`, `site_name`, `unit_id`, `unit_name`, `date`; default value is `date`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `desc`).

#### Response: `application/json`

```json
{
  "content": [
    {
      "event_id": "b6dcaff2-022e-461c-a4ca-01e822ccf3ac",
      "account_id": "0697b16e-ddcc-4711-b072-906fdcc05270",
      "account_name": "Test account",
      "site_id": "a77392d8-5241-44a2-84a0-c12c76a6b532",
      "site_name": "Test site",
      "unit_id": "77a063c4-00a7-452f-9856-e1999f1deb57",
      "unit_name": "unit123",
      "public": true,
      "type": "INFO",
      "title": "Mode change",
      "message": "Unit is in transportation mode now",
      "origin": {
        "app": "test",
        "code": 10,
        "level": "Info",
        "site_id": "a77392d8-5241-44a2-84a0-c12c76a6b532",
        "unit_id": "77a063c4-00a7-452f-9856-e1999f1deb57",
        "facility": "mode",
        "priority": "med",
        "site_name": "Test site",
        "unit_name": "unit123",
        "account_id": "0697b16e-ddcc-4711-b072-906fdcc05270",
        "description": "Unit is in transportation mode now",
        "account_name": "Test account",
        "capture_time": "2023-08-22T08:47:15.096Z",
        "receive_time": "2023-08-22T08:47:17.481210108Z"
      },
      "date": "2023-08-22T08:47:17.481210Z"
    },
    ...
  ],
  "page": 0,
  "page_size": 25,
  "total_pages": 290,
  "total_elements": 7233
}
```

## GET `/audit/dashboard/settings`

Get dashboard event generation rules (the rules that define dashboard event generation out of audit logs).

#### Access: `supervisor` having `system.management` permission

#### Request params/body: `none`

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
[
  {...}
]
```

*No dashboard settings:*

* Response code: `204 No Content`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "No dashboard settings"
    }
  ]
}
```

## PUT `/audit/dashboard/settings`

Update dashboard event generation rules.

#### Access: `supervisor` having `system.management` permission

#### Request body: `application/json`

For detailed rule format check [dashboard-rules] document.

```json
[
  {...}
]
```

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
[
  {...}
]
```

*No request body, or it's not a valid JSON:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Missing or invalid request body"
    }
  ]
}
```

*Validation failure:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "[2].@then[0]",
      "message": "Either @event or @then must be specified"
    }
  ]
}
```

# Management endpoints

## GET `/management/account`

Get account of authorized client (`unit` or `user`).

#### Access: `unit` or `user`, no permissions required

#### Request params/body: `none`

#### Response: `application/json`

```json
{
  "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
  "name": "Test account",
  "description": "Account created for demo",
  "creation_date": "2022-04-26T07:31:20.549917Z",
  "change_date": "2022-04-26T07:31:20.549917Z"
}
```

* `id` an identifier of an account;
* `name` a name of an account;
* `description` an optional description of an account;
* `creation_date` a timestamp of when an account was created;
* `change_date` a timestamp of when an account was changed the last time.

## GET `/management/account/{id}`

Get account by its identifier.

#### Access: `supervisor` having `account.view` permission

#### Request params:

* `{id}` an identifier of an account to get.

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
  "name": "Test account",
  "description": "Account created for demo",
  "creation_date": "2022-04-26T07:31:20.549917Z",
  "change_date": "2022-04-26T07:31:20.549917Z"
}
```

*Account not found:*

* Response code: `404 Not Found`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid account id: 0697b16e-ddcc-4711-b072-906fdcc05271"
    }
  ]
}
```

## GET `/management/accounts`

Get all accounts that the authorized client has access to.

#### Access: `supervisor` having `account.view` permission

#### Request params:

*Optional:*

* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `10`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `name`, `description`, `creation_date`
  and `change_date`; default value is `name`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `asc`).

#### Response: `application/json`

```json
{
  "content": [
    {
      "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
      "name": "Test account",
      "description": "Account created for demo",
      "creation_date": "2022-04-26T07:31:20.549917Z",
      "change_date": "2022-04-26T07:31:20.549917Z"
    },
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ],
  "page": 0,
  "page_size": 10,
  "total_pages": 3,
  "total_elements": 30
}
```

## POST `/management/account`

Create new account.

#### Access: `supervisor` having `account.create` permission

#### Request body: `application/json`

```json
{
  "name": "Test account",
  "description": "Account created for demo"
}
```

*Required fields:*

* `name` a name for a new account (3-255 characters).

*Optional fields:*

* `description` an optional description of an account (up to 10000 characters).

#### Response

*Success:*

A new account has been successfully created.

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
  "name": "Test account",
  "description": "Account created for demo",
  "creation_date": "2022-04-26T07:31:20.549917Z",
  "change_date": "2022-04-26T07:31:20.549917Z"
}
```

*Name is already in use:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "name",
      "message": "Name is already in use"
    }
  ]
}
```

## PUT `/management/account/{id}`

Change existing account by its identifier.

#### Access: `supervisor` having `account.edit` permission

#### Request params:

* `{id}` an identifier of an account to change.

#### Request body: `application/json`

```json
{
  "name": "Test account (altered name)",
  "description": "Account created for demo (altered description)"
}
```

#### Response

*Success:*

A new account has been successfully created.

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
  "name": "Test account",
  "description": "Account created for demo",
  "creation_date": "2022-04-26T07:31:20.549917Z",
  "change_date": "2022-04-26T07:31:20.549917Z"
}
```

*Name is already in use:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "name",
      "message": "Name is already in use"
    }
  ]
}
```

## DELETE `/management/account/{id}`

Delete account by its identifier.

#### Access: `supervisor` having `account.delete` permission

#### Request params:

* `{id}` an identifier of an account to delete.

#### Response

*Success:*

* Response code: `204 No Content`
* Response body: `none`

*There are users in this account:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "There are users in this account, you have to delete them first"
    }
  ]
}
```

*There are units depended on this account:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "There are units in this account, you have to delete them first"
    }
  ]
}
```

*There are sites depended on this account:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "There are sites in this account, you have to delete them first"
    }
  ]
}
```

## GET `/management/site`

Get site of authorized `unit`.

#### Access: `unit`, no permissions required

#### Request params/body: `none`

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "a77392d8-5241-44a2-84a0-c12c76a6b532",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "name": "Test site",
  "description": "Site created for demo",
  "creation_date": "2022-05-09T17:44:27.016319260Z",
  "change_date": "2022-05-09T17:44:27.016319260Z"
}
```

*Unit does not belong to any site:*

* Response code: `404 Not Found`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Unit is not registered at any site yet"
    }
  ]
}
```

## GET `/management/site/{id}`

Get site by its identifier.

#### Access: `user` or `supervisor` having `site.view` permission

#### Request params:

* `{id}` an identifier of a site to get.

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "a77392d8-5241-44a2-84a0-c12c76a6b532",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "name": "Test site",
  "description": "Site created for demo",
  "creation_date": "2022-05-09T17:44:27.016319Z",
  "change_date": "2022-05-09T17:44:27.016319Z"
}
```

*Site not found:*

* Response code: `404 Not Found`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid site id: a77392d8-5241-44a2-84a0-c12c76a6b533"
    }
  ]
}
```

## GET `/management/sites`

Get all sites that the authorized client has access to.

#### Access: `user` or `supervisor` having `site.view` permission

#### Request params:

*Optional:*

* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `10`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `name`, `description`, `creation_date`
  , `change_date`, `account_id` and `account_name`; default value is `name`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `asc`).

#### Response: `application/json`

```json
{
  "content": [
    {
      "id": "a77392d8-5241-44a2-84a0-c12c76a6b532",
      "account": {
        "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
        "name": "Test account"
      },
      "name": "Test site",
      "description": "Site created for demo",
      "creation_date": "2022-05-09T17:44:27.016319Z",
      "change_date": "2022-05-09T17:44:27.016319Z"
    },
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ],
  "page": 0,
  "page_size": 10,
  "total_pages": 3,
  "total_elements": 30
}
```

## GET `/management/account/{account_id}/sites`

Get referenced account's sites that the authorized client has access to.

#### Access: `supervisor` having `site.view` permission

#### Request params:

*Required:*

* `{account_id}` an identifier of an account to get sites of.

*Optional:*

* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `10`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `name`, `description`, `creation_date`
  , `change_date`, `account_id` and `account_name`; default value is `name`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `asc`).

#### Response: `application/json`

```json
{
  "content": [
    {
      "id": "a77392d8-5241-44a2-84a0-c12c76a6b532",
      "account": {
        "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
        "name": "Test account"
      },
      "name": "Test site",
      "description": "Site created for demo",
      "creation_date": "2022-05-09T17:44:27.016319Z",
      "change_date": "2022-05-09T17:44:27.016319Z"
    },
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ],
  "page": 0,
  "page_size": 10,
  "total_pages": 3,
  "total_elements": 30
}
```

## POST `/management/site`

Create new site. A `user` can only create a site within their own account. A `supervisor` must specify `account_id`
explicitly.

#### Access: `user` or `supervisor` having `site.create` permission (with proper account as permission target)

#### Request body: `application/json`

```json
{
  "account_id": "0697b16e-ddcc-4711-b072-906fdcc05270",
  "name": "Test site",
  "description": "Site created for demo"
}
```

*Required fields:*

* `name` a name for a new site (3-255 characters).

*Optional fields:*

* `account_id` an identifier of an account to create a new site within (this field is required for `supervisor` and
  ignored for `user`);
* `description` an optional description of a site (up to 10000 characters).

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "a77392d8-5241-44a2-84a0-c12c76a6b532",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "name": "Test site",
  "description": "Site created for demo",
  "creation_date": "2022-05-09T17:44:27.016319Z",
  "change_date": "2022-05-09T17:44:27.016319Z"
}
```

*Name is already in use:*

The site name must be unique within its account.

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "name",
      "message": "Name is already in use"
    }
  ]
}
```

*Account not found:*

If `supervisor` creates a new site and specifies invalid account identifier then the following error is responded.

* Response code: `404 Not Found`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid account id: 0697b16e-ddcc-4711-b072-906fdcc05271"
    }
  ]
}
```

## PUT `/management/site/{id}`

Change existing site by its identifier.

#### Access: `user` or `supervisor` having `site.edit` permission

#### Request params:

* `{id}` an identifier of a site to change.

#### Request body: `application/json`

```json
{
  "name": "Test site (altered name)",
  "description": "Site created for demo (altered description)"
}
```

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "a77392d8-5241-44a2-84a0-c12c76a6b532",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "name": "Test site (altered name)",
  "description": "Site created for demo (altered description)",
  "creation_date": "2022-05-09T17:44:27.016319Z",
  "change_date": "2022-05-09T18:10:51.742624Z"
}
```

*Name is already in use:*

The site name must be unique within its account.

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "name",
      "message": "Name is already in use"
    }
  ]
}
```

## DELETE `/management/site/{id}`

Delete site by its identifier.

#### Access: `user` or `supervisor` having `site.delete` permission

#### Request params:

* `{id}` an identifier of a site to delete.

#### Response

*Success:*

* Response code: `204 No Content`
* Response body: `none`

*There are units depended on this site:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "There are units in this site, you have to delete them first"
    }
  ]
}
```

## GET `/management/product/{id}`

Get product by its identifier.

#### Access: `user` or `supervisor`, no permissions required

#### Request params:

* `{id}` an identifier of a product to get.

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "960c1dd9-88d0-4a79-2ddb-4a5289a6cd90",
  "name": "BP216",
  "dimensions": "6-6-4",
  "description": "BOPIS 2 x 16",
  "creation_date": "2022-05-09T17:44:27.016319Z",
  "change_date": "2022-05-09T17:44:27.016319Z"
}
```

*Product not found:*

* Response code: `404 Not Found`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid product id: 960c1dd9-88d0-4a79-2ddb-4a5289a6cd91"
    }
  ]
}
```

## GET `/management/products`

Get all products ordered by `name` (case-insensitive).

#### Access: `user` or `supervisor`, no permissions required

#### Request params/body: `none`

#### Response: `application/json`

```json
[
  {
    "id": "960c1dd9-88d0-4a79-2ddb-4a5289a6cd90",
    "name": "BP216",
    "dimensions": "6-6-4",
    "description": "BOPIS 2 x 16",
    "creation_date": "2022-05-09T17:44:27.016319Z",
    "change_date": "2022-05-09T17:44:27.016319Z"
  },
  {
    ...
  },
  {
    ...
  },
  {
    ...
  }
]
```

## POST `/management/product`

Create new product.

#### Access: `supervisor` having `system.management` permission

#### Request body: `application/json`

```json
{
  "name": "BP216",
  "dimensions": "6-6-4",
  "description": "BOPIS 2 x 16"
}
```

*Required fields:*

* `name` a unique name of a product (2-64 characters);
* `dimensions` a physical layout of a product, must match regex `[1-6](-[1-6])*`.

*Optional fields:*

* `description` a description of a product (up to 255 characters).

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "960c1dd9-88d0-4a79-2ddb-4a5289a6cd90",
  "name": "BP216",
  "dimensions": "6-6-4",
  "description": "BOPIS 2 x 16",
  "creation_date": "2022-05-09T17:44:27.016319Z",
  "change_date": "2022-05-09T17:44:27.016319Z"
}
```

*Name is already in use:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "name",
      "message": "Name is already in use"
    }
  ]
}
```

## PUT `/management/product/{id}`

Change existing product by its identifier.

#### Access: `supervisor` having `system.management` permission

#### Request params:

* `{id}` an identifier of a product to change.

#### Request body: `application/json`

```json
{
  "name": "BP216",
  "dimensions": "6-6-5",
  "description": "BOPIS 2 x 16"
}
```

*Required fields:*

* `name` a unique name of a product (2-64 characters);
* `dimensions` a physical layout of a product, must match regex `[1-6](-[1-6])*`.

*Optional fields:*

* `description` a description of a product (up to 255 characters).

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "960c1dd9-88d0-4a79-2ddb-4a5289a6cd90",
  "name": "BP216",
  "dimensions": "6-6-4",
  "description": "BOPIS 2 x 16",
  "creation_date": "2022-05-09T17:44:27.016319Z",
  "change_date": "2022-05-09T18:10:51.742624Z"
}
```

*Name is already in use:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "name",
      "message": "Name is already in use"
    }
  ]
}
```

*Cannot change dimensions of a product used by registered unit(s):*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "dimensions",
      "message": "There are registered units bound to this product, you cannot change dimensions of a product which is already in use. Unregister all the related units first"
    }
  ]
}
```

## DELETE `/management/product/{id}`

Delete product by its identifier.

#### Access: `supervisor` having `system.management` permission

#### Request params:

* `{id}` an identifier of a product to delete.

#### Response

*Success:*

* Response code: `204 No Content`
* Response body: `none`

*Product is in use:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "There are units bound to this product, you have to delete/change them first"
    }
  ]
}
```

## GET `/management/firmware_categories`

Get all available firmware categories.

#### Access: `user` or `supervisor`, no permissions required

#### Request params/body: `none`

#### Response: `application/json`

```json
[
  {
    "id": "9871f095-89e6-444a-81c3-73b723cd8e55",
    "name": "foo",
    "creation_date": "2023-07-20T08:12:50.944916Z",
    "change_date": "2023-07-20T08:12:50.944916Z"
  },
  {
    "id": "28456545-4449-4c55-a611-2a863786c184",
    "name": "bar",
    "creation_date": "2023-07-20T08:13:05.640749Z",
    "change_date": "2023-07-20T08:13:05.640749Z"
  },
  {
    "id": "eb2ea131-0b5e-4418-94c4-7fd4d7d05c69",
    "name": "xyz",
    "creation_date": "2023-07-20T08:13:09.407776Z",
    "change_date": "2023-07-20T08:13:27.868776Z"
  }
]
```

## POST `/management/firmware_category`

Create new firmware category.

#### Access: `supervisor` having `system.management` permission

#### Request body: `application/json`

```json
{
  "name": "foo"
}
```

*Required fields:*

* `name` a name for a new firmware category (2-64 characters).

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "9871f095-89e6-444a-81c3-73b723cd8e55",
  "name": "foo",
  "creation_date": "2023-07-20T08:12:50.944916419Z",
  "change_date": "2023-07-20T08:12:50.944916419Z"
}
```

*Name is already in use:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "name",
      "message": "Name is already in use"
    }
  ]
}
```

## PUT `/management/firmware_category/{id}`

Change existing firmware category by its identifier.

#### Access: `supervisor` having `system.management` permission

#### Request params:

* `{id}` an identifier of a firmware category to change.

#### Request body: `application/json`

```json
{
  "name": "xyz"
}
```

*Required fields:*

* `name` a new name for a firmware category (2-64 characters).

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "eb2ea131-0b5e-4418-94c4-7fd4d7d05c69",
  "name": "xyz",
  "creation_date": "2023-07-20T08:13:09.407776Z",
  "change_date": "2023-07-20T08:13:27.868776307Z"
}
```

*Name is already in use:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "name",
      "message": "Name is already in use"
    }
  ]
}
```

*Firmware category not found:*

* Response code: `404 Not Found`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid firmware category id: eb2ea131-0b5e-4418-94c4-7fd4d7d05c68"
    }
  ]
}
```

## DELETE `/management/firmware_category/{id}`

Delete firmware category by its identifier.

#### Access: `supervisor` having `system.management` permission

#### Request params:

* `{id}` an identifier of a firmware category to delete.

#### Response

*Success:*

* Response code: `204 No Content`
* Response body: `none`

*Firmware category is in use:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Cannot delete firmware category in use"
    }
  ]
}
```

*Firmware category not found:*

* Response code: `404 Not Found`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid firmware category id: eb2ea131-0b5e-4418-94c4-7fd4d7d05c68"
    }
  ]
}
```

## GET `/management/unit`

Get current unit.

#### Access: `unit`, no permissions required

#### Request params/body: `none`

#### Response: `application/json`

```json
{
  "id": "77a063c4-00a7-452f-9856-e1999f1deb57",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "site": null,
  "name": "unit123",
  "description": "Unit created for demo",
  "registration_state": "UNREGISTERED",
  "product": {
    "id": "f8d3fdd3-a13d-ab22-6f2c-d77fb0082268",
    "name": "T211",
    "dimensions": "5-6",
    "description": "CellVault 2 x 11"
  },
  "firmware_category": {
    "id": "59e8a55c-c97b-11ed-afa1-0242ac120002",
    "name": "cellvault"
  },
  "access_cards": [
    {
      "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
      "name": "card-12345"
    }
  ],
  "creation_date": "2022-04-26T07:36:27.218080Z",
  "change_date": "2022-04-26T07:36:27.218080Z"
}
```

## GET `/management/unit/{id}`

Get unit by its identifier.

#### Access: `user` or `supervisor` having `unit.view` permission

#### Request params:

* `{id}` an identifier of a unit to get.

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "77a063c4-00a7-452f-9856-e1999f1deb57",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "site": null,
  "name": "unit123",
  "description": "Unit created for demo",
  "registration_state": "UNREGISTERED",
  "product": {
    "id": "f8d3fdd3-a13d-ab22-6f2c-d77fb0082268",
    "name": "T211",
    "dimensions": "5-6",
    "description": "CellVault 2 x 11"
  },
  "firmware_category": {
    "id": "59e8a55c-c97b-11ed-afa1-0242ac120002",
    "name": "cellvault"
  },
  "access_cards": [
    {
      "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
      "name": "card-12345"
    }
  ],
  "creation_date": "2022-04-26T07:36:27.218080Z",
  "change_date": "2022-04-26T07:36:27.218080Z"
}
```

*Unit not found:*

* Response code: `404 Not Found`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid unit id: 0697b16e-ddcc-4711-b072-906fdcc05271"
    }
  ]
}
```

## GET `/management/unit/by-name/{name}`

Get unit by its name.

#### Access: `user` or `supervisor` having `unit.view` permission

#### Request params:

* `{name}` a name of a unit to get.

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "77a063c4-00a7-452f-9856-e1999f1deb57",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "site": null,
  "name": "unit123",
  "description": "Unit created for demo",
  "registration_state": "UNREGISTERED",
  "product": {
    "id": "f8d3fdd3-a13d-ab22-6f2c-d77fb0082268",
    "name": "T211",
    "dimensions": "5-6",
    "description": "CellVault 2 x 11"
  },
  "firmware_category": {
    "id": "59e8a55c-c97b-11ed-afa1-0242ac120002",
    "name": "cellvault"
  },
  "access_cards": [
    {
      "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
      "name": "card-12345"
    }
  ],
  "creation_date": "2022-04-26T07:36:27.218080Z",
  "change_date": "2022-04-26T07:36:27.218080Z"
}
```

*Unit not found:*

* Response code: `404 Not Found`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid unit name: unit123"
    }
  ]
}
```

## GET `/management/unit/settings`

Get current unit's settings. If `since` request parameter (date) is specified then the response will be `HTTP 204` (with
empty body) unless there are some changes to settings since `since` date.

#### Access: `unit`, no permissions required

#### Request params:

*Optional:*

* `since` if specified then the response will be `HTTP 204` unless last change date of the settings is greater
  than `since` value.

#### Response

*Settings changed or `since` is not provided:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "access_cards_settings": {
    "cards": [
      {
        "user_id": "62956ac1-52d3-4990-8156-f8fd22962727",
        "full_name": "John Doe",
        "code": "12345",
        "pin": null
      },
      {
        "user_id": "62956ac1-52d3-4990-8156-f8fd22962727",
        "full_name": "John Doe",
        "code": "99999",
        "pin": "159357"
      }
    ],
    "change_date": "2022-06-10T14:43:23.006310787Z"
  },
  "mqtt_settings": {
    "server_uri": "mqtt://18.222.99.94:1883",
    "password": "p@$$w0rd",
    "username": "cv12345",
    "change_date": "2022-06-10T13:21:53.456Z"
  },
  "change_date": "2022-06-10T14:43:23.006311Z"
}
```

*Settings didn't change since `since` date*

* Response code: `204 No Content`
* Response body: `none`

## GET `/management/units`

Get all units that the authorized client has access to.

#### Access: `user` or `supervisor` having `unit.view` permission

#### Request params:

*Optional:*

* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `10`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `name`, `description`, `registration_state`
  , `product_name`, `product_dimensions`, `product_description`, `creation_date`, `change_date`, `account_id`
  , `account_name`, `site_id` and `site_name`; default value is `name`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `asc`).

#### Response: `application/json`

```json
{
  "content": [
    {
      "id": "77a063c4-00a7-452f-9856-e1999f1deb57",
      "account": {
        "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
        "name": "Test account"
      },
      "site": null,
      "name": "unit123",
      "description": "Unit created for demo",
      "registration_state": "UNREGISTERED",
      "product": {
        "id": "f8d3fdd3-a13d-ab22-6f2c-d77fb0082268",
        "name": "T211",
        "dimensions": "5-6",
        "description": "CellVault 2 x 11"
      },
      "firmware_category": {
        "id": "59e8a55c-c97b-11ed-afa1-0242ac120002",
        "name": "cellvault"
      },
      "access_cards": [
        {
          "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
          "name": "card-12345"
        }
      ],
      "creation_date": "2022-04-26T07:36:27.218080Z",
      "change_date": "2022-04-26T07:36:27.218080Z"
    },
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ],
  "page": 0,
  "page_size": 10,
  "total_pages": 3,
  "total_elements": 30
}
```

## GET `/management/site/{site_id}/units`

Get referenced site's units that the authorized client has access to.

#### Access: `user` or `supervisor` having `unit.view` permission

#### Request params:

*Required:*

* `{site_id}` an identifier of a site to get units of.

*Optional:*

* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `10`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `name`, `description`, `registration_state`
  , `product_name`, `product_dimensions`, `product_description`, `creation_date`, `change_date`, `account_id`
  , `account_name`, `site_id` and `site_name`; default value is `name`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `asc`).

#### Response: `application/json`

```json
{
  "content": [
    {
      "id": "77a063c4-00a7-452f-9856-e1999f1deb57",
      "account": {
        "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
        "name": "Test account"
      },
      "site": {
        "id": "a77392d8-5241-44a2-84a0-c12c76a6b532",
        "name": "Test site"
      },
      "name": "unit123",
      "description": "Unit created for demo",
      "registration_state": "REGISTERED",
      "product": {
        "id": "f8d3fdd3-a13d-ab22-6f2c-d77fb0082268",
        "name": "T211",
        "dimensions": "5-6",
        "description": "CellVault 2 x 11"
      },
      "firmware_category": {
        "id": "59e8a55c-c97b-11ed-afa1-0242ac120002",
        "name": "cellvault"
      },
      "access_cards": [
        {
          "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
          "name": "card-12345"
        }
      ],
      "creation_date": "2022-04-26T07:36:27.218080Z",
      "change_date": "2022-05-09T21:42:11.712862Z"
    },
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ],
  "page": 0,
  "page_size": 10,
  "total_pages": 3,
  "total_elements": 30
}
```

## GET `/management/account/{account_id}/units`

Get referenced account's units that the authorized client has access to.

#### Access: `supervisor` having `unit.view` permission

#### Request params:

*Required:*

* `{account_id}` an identifier of an account to get units of.

*Optional:*

* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `10`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `name`, `description`, `registration_state`
  , `product_name`, `product_dimensions`, `product_description`, `creation_date`, `change_date`, `account_id`
  , `account_name`, `site_id` and `site_name`; default value is `name`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `asc`).

#### Response: `application/json`

```json
{
  "content": [
    {
      "id": "77a063c4-00a7-452f-9856-e1999f1deb57",
      "account": {
        "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
        "name": "Test account"
      },
      "site": null,
      "name": "unit123",
      "description": "Unit created for demo",
      "registration_state": "UNREGISTERED",
      "product": {
        "id": "f8d3fdd3-a13d-ab22-6f2c-d77fb0082268",
        "name": "T211",
        "dimensions": "5-6",
        "description": "CellVault 2 x 11"
      },
      "firmware_category": {
        "id": "59e8a55c-c97b-11ed-afa1-0242ac120002",
        "name": "cellvault"
      },
      "access_cards": [
        {
          "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
          "name": "card-12345"
        }
      ],
      "creation_date": "2022-04-26T07:36:27.218080Z",
      "change_date": "2022-04-26T07:36:27.218080Z"
    },
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ],
  "page": 0,
  "page_size": 10,
  "total_pages": 3,
  "total_elements": 30
}
```

## GET `/management/card/{card_id}/accessible-units`

Get units that the referenced access card is attached to.

#### Access: `user` or `supervisor` having `unit.view` permission for units and `card.view` permission for referenced access card (`card.view` permission is not required for `user` who is the owner of access card)

#### Request params:

*Required:*

* `{card_id}` an identifier of an access card to get accessible units for.

*Optional:*

* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `10`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `name`, `description`, `registration_state`
  , `product_name`, `product_dimensions`, `product_description`, `creation_date`, `change_date`, `account_id`
  , `account_name`, `site_id` and `site_name`; default value is `name`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `asc`).

#### Response: `application/json`

```json
{
  "content": [
    {
      "id": "77a063c4-00a7-452f-9856-e1999f1deb57",
      "account": {
        "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
        "name": "Test account"
      },
      "site": null,
      "name": "unit123",
      "description": "Unit created for demo",
      "registration_state": "UNREGISTERED",
      "product": {
        "id": "f8d3fdd3-a13d-ab22-6f2c-d77fb0082268",
        "name": "T211",
        "dimensions": "5-6",
        "description": "CellVault 2 x 11"
      },
      "firmware_category": {
        "id": "59e8a55c-c97b-11ed-afa1-0242ac120002",
        "name": "cellvault"
      },
      "access_cards": [
        {
          "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
          "name": "card-12345"
        }
      ],
      "creation_date": "2022-04-26T07:36:27.218080Z",
      "change_date": "2022-04-26T07:36:27.218080Z"
    },
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ],
  "page": 0,
  "page_size": 10,
  "total_pages": 3,
  "total_elements": 30
}
```

## POST `/management/unit`

Create new unit.

#### Access: `supervisor` having `unit.create` permission

#### Request body: `application/json`

```json
{
  "account_id": "0697b16e-ddcc-4711-b072-906fdcc05270",
  "name": "unit123",
  "description": "Unit created for demo",
  "product_id": "f8d3fdd3-a13d-ab22-6f2c-d77fb0082268",
  "firmware_category_id": "59e8a55c-c97b-11ed-afa1-0242ac120002"
}
```

*Required fields:*

* `account_id` an identifier of an account to create a new unit within;
* `name` a name for a new unit (3-255 characters);
* `product_id` an identifier of a product that describes the unit;
* `firmware_category_id` an identifier of a firmware category that the unit belongs to.

*Optional fields:*

* `description` an optional description of a unit (up to 10000 characters).

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "77a063c4-00a7-452f-9856-e1999f1deb57",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "site": null,
  "name": "unit123",
  "description": "Unit created for demo",
  "registration_state": "UNREGISTERED",
  "product": {
    "id": "f8d3fdd3-a13d-ab22-6f2c-d77fb0082268",
    "name": "T211",
    "dimensions": "5-6",
    "description": "CellVault 2 x 11"
  },
  "firmware_category": {
    "id": "59e8a55c-c97b-11ed-afa1-0242ac120002",
    "name": "cellvault"
  },
  "access_cards": [],
  "creation_date": "2022-04-26T07:36:27.218080Z",
  "change_date": "2022-04-26T07:36:27.218080Z"
}
```

*Name is already in use:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "name",
      "message": "Name is already in use"
    }
  ]
}
```

## PUT `/management/unit/{id}`

Change existing unit by its identifier.

#### Access: `user` or `supervisor` having `unit.edit` permission

#### Request params:

* `{id}` an identifier of a unit to change.

#### Request body: `application/json`

```json
{
  "description": "Unit created for demo (altered description)",
  "product_id": "f8d3fdd3-a13d-ab22-6f2c-d77fb0082268",
  "firmware_category_id": "59e8a55c-c97b-11ed-afa1-0242ac120002"
}
```

*Optional fields:*

* `description` an optional description of a unit (up to 10000 characters);
* `product_id` an identifier of a product that describes the unit;
* `firmware_category_id` an identifier of a firmware category that the unit belongs to.

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "77a063c4-00a7-452f-9856-e1999f1deb57",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "site": null,
  "name": "unit123",
  "description": "Unit created for demo (altered description)",
  "registration_state": "UNREGISTERED",
  "product": {
    "id": "f8d3fdd3-a13d-ab22-6f2c-d77fb0082268",
    "name": "T211",
    "dimensions": "5-6",
    "description": "CellVault 2 x 11"
  },
  "firmware_category": {
    "id": "59e8a55c-c97b-11ed-afa1-0242ac120002",
    "name": "cellvault"
  },
  "access_cards": [],
  "creation_date": "2022-04-26T07:36:27.218080Z",
  "change_date": "2022-04-27T09:12:44.348362Z"
}
```

*Cannot change product of registered unit:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "You cannot bind a unit to another product while it's registered"
    }
  ]
}
```

## GET `/management/unit/{id}/registration`

Get registration state and (if any) registration request information for referenced unit.

#### Access: `supervisor` having `unit.registration` permission

#### Request params:

* `{id}` an identifier of a unit to get registration information of.

#### Response: `application/json`

*Success (unit is in unregistered state):*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "state": "UNREGISTERED"
}
```

*Success (unit is scheduled for registration):*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "state": "CAN_REGISTER",
  "request": {
    "code": "93952791",
    "creation_date": "2022-05-09T19:37:45.851166176Z",
    "expiration_date": "2022-05-10T19:37:45.851166176Z"
  }
}
```

*Success (unit is in registered state):*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "state": "REGISTERED"
}
```

## POST `/management/unit/{id}/registration/schedule`

Submit registration request (schedule registration) for referenced unit.

#### Access: `supervisor` having `unit.registration` permission

#### Request params:

* `{id}` an identifier of a unit to schedule registration for.

#### Request body: `application/json`

```json
{
  "site_id": "a77392d8-5241-44a2-84a0-c12c76a6b532"
}
```

*Required fields:*

* `site_id` an identifier of a site where the referenced unit will be installed.

#### Response

*Success (unit is scheduled for registration now):*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "state": "CAN_REGISTER",
  "request": {
    "code": "93952791",
    "creation_date": "2022-05-09T19:37:45.851166176Z",
    "expiration_date": "2022-05-10T19:37:45.851166176Z"
  }
}
```

*Unit is already registered:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Unit is already registered"
    }
  ]
}
```

*An attempt to register the unit in a site which does not belong to unit's account:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "You cannot schedule registration of a unit at a site which does not belong to unit's account"
    }
  ]
}
```

## POST `/management/unit/{id}/registration/reset`

Reset registration of referenced unit: change unit's registration state to unregistered and remove registration request
if any.

#### Access: `supervisor` having `unit.registration` permission

#### Request params:

* `{id}` an identifier of a unit to reset registration for.

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "state": "UNREGISTERED"
}
```

*Unit is in unregistered state:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Unit is already unregistered"
    }
  ]
}
```

## POST `/management/unit/{id}/cards/attach`

Attach referenced unit to given access card(s).

#### Access: `user` or `supervisor` having `card.assign` permission for a unit and `card.edit` permission for the cards

#### Request params:

* `{id}` an identifier of a unit to attach access card(s) to.

#### Request body: `application/json`

```json
{
  "cards": [
    "2b452f16-d9a0-4438-a7e0-da1e73b08041",
    ...
  ]
}
```

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `none`

*Card is not assigned:*

* Response code: `400 OK`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "cards",
      "message": "You must assign a card to some account to be able to attach it to a unit"
    }
  ]
}
```

*Card is assigned to a different account:*

* Response code: `400 OK`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "cards",
      "message": "You can only attach a card to a unit within the same account"
    }
  ]
}
```

## POST `/management/unit/{id}/cards/detach`

Detach referenced unit from given access card(s).

#### Access: `user` or `supervisor` having `card.assign` permission for a unit and `card.edit` permission for the cards

#### Request params:

* `{id}` an identifier of a unit to detach access card(s) from.

#### Request body: `application/json`

```json
{
  "cards": [
    "2b452f16-d9a0-4438-a7e0-da1e73b08041",
    ...
  ]
}
```

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `none`

## DELETE `/management/unit/{id}`

Delete unit by its identifier.

#### Access: `supervisor` having `unit.delete` permission

#### Request params:

* `{id}` an identifier of a unit to delete.

#### Response

*Success:*

* Response code: `204 No Content`
* Response body: `none`

*Unit is registered or scheduled for registration:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Cannot delete unit. You have to set it to unregistered state first"
    }
  ]
}
```

## POST `/management/unit/{id}/exchange_mqtt`

Send an MQTT message to unit by its identifier.

#### Access: `user` or `supervisor` having `unit.mqtt` permission

#### Request params:

* `{id}` an identifier of a unit to send a message to.

#### Request body: `application/json`

```json
{
  "function": "do-something",
  "body": "something",
  "timeout": 10000
}
```

*Required fields:*

* `function` a name of a built-in or a custom client-specific function name;
* `timeout` a time period in milliseconds which specifies for how long the server will wait for a response from a unit (between 100 and 60000).

*Optional fields:*

* `body` some extra payload (up to 512 characters).

#### Response

*Success (the unit processed the message successfully and the workflow was completed):*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "status": "OK",
  "msg_body": null
}
```

*Timeout (no response from a unit, it's likely to be offline):*

* Response code: `502 Bad Gateway`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "No response from unit"
    }
  ]
}
```

*Failure (the unit did not process the message successfully, but the workflow was completed):*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "status": "FAILED",
  "msg_body": "Something went wrong"
}
```

## GET `/management/user`

Get current user/supervisor.

#### Access: `user` or `supervisor`, no permissions required

#### Request params/body: `none`

#### Response: `application/json`

```json
{
  "id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "username": "john.doe@vaultgroup.co.za",
  "full_name": "John Doe",
  "active": true,
  "creation_date": "2022-05-09T09:14:38.760430915Z",
  "change_date": "2022-05-09T09:22:12.345346322Z"
}
```

## GET `/management/user/{id}`

Get user/supervisor by its identifier.

#### Access: `user` or `supervisor` having `user.view` permission

#### Request params:

* `{id}` an identifier of a user/supervisor to get.

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "username": "john.doe@vaultgroup.co.za",
  "full_name": "John Doe",
  "active": false,
  "invitation": {
    "creation_date": "2022-05-09T09:14:38.760430915Z",
    "expiration_date": "2022-05-10T09:14:38.760430915Z"
  },
  "creation_date": "2022-05-09T09:14:38.760430915Z",
  "change_date": "2022-05-09T09:22:12.345346322Z"
}
```

*User/supervisor not found:*

* Response code: `404 Not Found`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid user id: 32af44d2-ae7d-44e8-9eb4-38580c3da77d"
    }
  ]
}
```

## GET `/management/users`

Get all users/supervisors that the authorized client has access to.

#### Access: `user` or `supervisor` having `user.view` permission

#### Request params:

*Optional:*

* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `10`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `username`, `full_name`, `active`
  , `creation_date`, `change_date`, `account_id` and `account_name`; default value is `username`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `asc`).

#### Response: `application/json`

```json
{
  "content": [
    {
      "id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c",
      "account": {
        "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
        "name": "Test account"
      },
      "username": "john.doe@vaultgroup.co.za",
      "full_name": "John Doe",
      "active": true,
      "creation_date": "2022-05-09T09:14:38.760430915Z",
      "change_date": "2022-05-09T09:22:12.345346322Z"
    },
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ],
  "page": 0,
  "page_size": 10,
  "total_pages": 3,
  "total_elements": 30
}
```

## GET `/management/account/{account_id}/users`

Get referenced account's users (supervisors do not belong to any account) that the authorized client has access to.

#### Access: `supervisor` having `user.view` permission

#### Request params:

*Required:*

* `{account_id}` an identifier of an account to get users of.

*Optional:*

* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `10`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `username`, `full_name`, `active`
  , `creation_date`, `change_date`, `account_id` and `account_name`; default value is `username`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `asc`).

#### Response: `application/json`

```json
{
  "content": [
    {
      "id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c",
      "account": {
        "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
        "name": "Test account"
      },
      "username": "john.doe@vaultgroup.co.za",
      "full_name": "John Doe",
      "active": true,
      "creation_date": "2022-05-09T09:14:38.760430915Z",
      "change_date": "2022-05-09T09:22:12.345346322Z"
    },
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ],
  "page": 0,
  "page_size": 10,
  "total_pages": 3,
  "total_elements": 30
}
```

## POST `/management/user`

Create new user/supervisor.

An e-mail with invitation link/code will be sent to the user.

When creating a new user you must specify its initial permissions (see [security] document). To adjust permissions
afterward (see `PUT /management/user/{id}/permissions` endpoint) you will need `user.permissions.edit` permission with
proper targets.

#### Access: `user` or `supervisor` having `user.create` permission

To create user within certain account you need that account to be a permission target of `user.create` permission.
To create supervisor you need `urn:*` as a permission target of `user.create` permission.

#### Request body: `application/json`

```json
{
  "account_id": "0697b16e-ddcc-4711-b072-906fdcc05270",
  "username": "John.Doe@VaultGroup.Co.Za",
  "full_name": "John Doe",
  "permissions": [
    {
      "tokens": [
        "unit.view",
        "unit.edit"
      ],
      "target_urns": [
        "urn:*"
      ]
    }
  ]
}
```

*Required fields:*

* `username` a unique name of the new user (will be converted to lower case during saving);
* `permissions` the permissions to be granted to the new user. You cannot give new user an access wider than you have so
  all the specified permissions and permission targets will be validated and automatically erased to meet that
  requirement.

*Optional fields:*

* `account_id` an identifier of an account to create user within (omit to create a supervisor);
* `full_name` a full name of a user, can be used as a description (up to 255 characters).

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "username": "john.doe@vaultgroup.co.za",
  "full_name": "John Doe",
  "active": false,
  "invitation": {
    "code": "98gTlqbHCi9H9gXhaIOhVdGJnb4OTKJQnZCHNrAD1ZfOm89Iguev9YUbMrRkqZ43",
    "creation_date": "2022-05-09T09:14:38.760430915Z",
    "expiration_date": "2022-05-10T09:14:38.760430915Z"
  },
  "creation_date": "2022-05-09T09:14:38.760430915Z",
  "change_date": "2022-05-09T09:14:38.760430915Z"
}
```

Notice that `invitation.code` property is only available on user creation. It will not be available for reading
afterwards.

*Name is already in use:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "name",
      "message": "Name is already in use"
    }
  ]
}
```

## PUT `/management/user/{id}`

Change existing user/supervisor by its identifier.

#### Access: `user` or `supervisor` having `user.edit` permission

#### Request params:

* `{id}` an identifier of a user/supervisor to change.

#### Request body: `application/json`

```json
{
  "full_name": "John Smith"
}
```

#### Response: `application/json`

```json
{
  "id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "username": "john.doe@vaultgroup.co.za",
  "full_name": "John Smith",
  "active": false,
  "invitation": {
    "creation_date": "2022-05-09T09:14:38.760430915Z",
    "expiration_date": "2022-05-10T09:14:38.760430915Z"
  },
  "creation_date": "2022-05-09T09:14:38.760430915Z",
  "change_date": "2022-05-10T12:48:33.827529461Z"
}
```

*Optional fields:*

* `full_name` a full name of a user, can be used as a description (up to 255 characters).

## GET `/management/user/{id}/permissions`

Get permissions of referenced user/supervisor.

#### Access: `user` or `supervisor` having `user.permissions.edit` permission

#### Request params:

* `{id}` an identifier of a user/supervisor to get permissions of.

#### Response: `application/json`

```json
[
  {
    "tokens": [
      "account.view",
      "account.edit",
      "account.create",
      "account.delete"
    ],
    "target_urns": [
      "urn:*"
    ]
  },
  {
    "tokens": [
      "unit.view",
      "unit.edit"
    ],
    "target_urns": [
      "urn:account/0697b16e-ddcc-4711-b072-906fdcc05270"
    ]
  }
]
```

## PUT `/management/user/{id}/permissions`

Change permissions of referenced user/supervisor.

You cannot give a user an access wider that you have so all the specified permissions and permission targets will be
validated and automatically erased to meet that requirement.

#### Access: `user` or `supervisor` having `user.permissions.edit` permission

#### Request body: `application/json`

```json
[
  {
    "tokens": [
      "account.view",
      "account.edit",
      "account.create",
      "account.delete"
    ],
    "target_urns": [
      "urn:*"
    ]
  },
  {
    "tokens": [
      "unit.view",
      "unit.edit"
    ],
    "target_urns": [
      "urn:account/0697b16e-ddcc-4711-b072-906fdcc05270"
    ]
  }
]
```

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
[
  {
    "tokens": [
      "account.view",
      "account.edit",
      "account.create",
      "account.delete"
    ],
    "target_urns": [
      "urn:*"
    ]
  },
  {
    "tokens": [
      "unit.view",
      "unit.edit"
    ],
    "target_urns": [
      "urn:account/0697b16e-ddcc-4711-b072-906fdcc05270"
    ]
  }
]
```

*Attempt to change own permissions:*

* Response code: `403 Forbidden`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "You cannot change your own permissions"
    }
  ]
}
```

*Invalid permission token:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "permissions",
      "message": "Invalid permission token: `universe.govern`"
    }
  ]
}
```

*Invalid permission target:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "permissions",
      "message": "Invalid urn format: `urn:foo/123`"
    }
  ]
}
```

## POST `/management/user/{id}/invitation/refresh`

Generate another invitation for a new user that hasn't signed up yet. Invitation expires in some time after user has
been created so use this endpoint to refresh an expired invitation.
A new e-mail with invitation link/code will be sent to the user.

#### Access: `user` or `supervisor` having `user.create` permission

The only user who can refresh invitation is the one who created that user and sent initial invitation.

#### Request params:

* `{id}` an identifier of a user/supervisor to generate a new invitation for.

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "code": "fZUkvxjw5FgTR2dcAjiVt9BKB8ysot5VGFNfFhrckyuENNVBGXPgKLS1Ouva4k0q",
  "creation_date": "2022-05-13T20:50:20.910236927Z",
  "expiration_date": "2022-05-20T20:50:20.910236927Z"
}
```

*User is already activated:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "User is already activated"
    }
  ]
}
```

*You're not the one who created this user:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "You cannot refresh invitation of a user created by someone else"
    }
  ]
}
```

## DELETE `/management/user/{id}`

Delete user/supervisor by its identifier.

#### Access: `user` or `supervisor` having `user.delete` permission

#### Request params:

* `{id}` an identifier of a user/supervisor to delete.

#### Response

*Success:*

* Response code: `204 No Content`
* Response body: `none`

## GET `/management/card/{id}`

Get access card by its identifier.

#### Access: `user` or `supervisor` having `card.view` permission (not required for `user` who is the owner of access card)

#### Request params:

* `{id}` an identifier of an access card to get.

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "user": {
    "id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c",
    "name": "john.doe@vaultgroup.co.za"
  },
  "name": "card-12345",
  "description": "The test card",
  "creation_date": "2022-06-09T11:26:13.289095Z",
  "change_date": "2022-06-09T17:22:36.708048Z"
}
```

*Card not found:*

* Response code: `404 Not Found`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid card id: 0697b16e-ddcc-4711-b072-906fdcc05271"
    }
  ]
}
```

## GET `/management/card/by-code/{code}`

Get access card by its code.

#### Access: `user` or `supervisor` having `card.view` permission (not required for `user` who is the owner of access card)

#### Request params:

* `{code}` a unique code of an access card to get.

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "user": {
    "id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c",
    "name": "john.doe@vaultgroup.co.za"
  },
  "name": "card-123456789",
  "description": "The test card",
  "creation_date": "2022-06-09T11:26:13.289095Z",
  "change_date": "2022-06-09T17:22:36.708048Z"
}
```

*Card not found:*

* Response code: `404 Not Found`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid card code: 111111111"
    }
  ]
}
```

## GET `/management/cards`

Get all access cards that the authorized client has access to.

#### Access: `user` or `supervisor` having `card.view` permission (not required for `user` who is the owner of access card)

#### Request params:

*Optional:*

* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `10`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `name`, `description`, `creation_date`
  , `change_date`, `user_id`, `user_name`, `account_id` and `account_name`; default value is `name`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `asc`).

#### Response: `application/json`

```json
{
  "content": [
    {
      "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
      "account": {
        "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
        "name": "Test account"
      },
      "user": {
        "id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c",
        "name": "john.doe@vaultgroup.co.za"
      },
      "name": "card-123456789",
      "description": "The test card",
      "creation_date": "2022-06-09T11:26:13.289095Z",
      "change_date": "2022-06-09T17:22:36.708048Z",
      "pin_assigned": false
    },
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ],
  "page": 0,
  "page_size": 10,
  "total_pages": 3,
  "total_elements": 30
}
```

## GET `/management/account/{account_id}/cards`

Get referenced account's access cards (see `POST /management/card/{id}/assign` and `POST /management/card/{id}/revoke`
endpoints) that the authorized client has access to.

#### Access: `supervisor` having `account.view` permission for an account and `card.view` permission for access cards

#### Request params:

*Optional:*

* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `10`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `name`, `description`, `creation_date`
  , `change_date`, `user_id`, `user_name`, `account_id` and `account_name`; default value is `name`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `asc`).

#### Response: `application/json`

```json
{
  "content": [
    {
      "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
      "account": {
        "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
        "name": "Test account"
      },
      "user": {
        "id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c",
        "name": "john.doe@vaultgroup.co.za"
      },
      "name": "card-123456789",
      "description": "The test card",
      "creation_date": "2022-06-09T11:26:13.289095Z",
      "change_date": "2022-06-09T17:22:36.708048Z",
      "pin_assigned": false
    },
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ],
  "page": 0,
  "page_size": 10,
  "total_pages": 3,
  "total_elements": 30
}
```

## GET `/management/user/{user_id}/cards`

Get referenced user's access cards (see `POST /management/card/{id}/assign` and `POST /management/card/{id}/revoke`
endpoints) that the authorized client has access to.

#### Access: `user` or `supervisor` having `user.view` permission for a user and `card.view` permission for access cards (`card.view` permission is not required for `user` who is the owner of access card)

#### Request params:

*Optional:*

* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `10`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `name`, `description`, `creation_date`
  , `change_date`, `user_id`, `user_name`, `account_id` and `account_name`; default value is `name`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `asc`).

#### Response: `application/json`

```json
{
  "content": [
    {
      "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
      "account": {
        "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
        "name": "Test account"
      },
      "user": {
        "id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c",
        "name": "john.doe@vaultgroup.co.za"
      },
      "name": "card-123456789",
      "description": "The test card",
      "creation_date": "2022-06-09T11:26:13.289095Z",
      "change_date": "2022-06-09T17:22:36.708048Z",
      "pin_assigned": false
    },
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ],
  "page": 0,
  "page_size": 10,
  "total_pages": 3,
  "total_elements": 30
}
```

## GET `/management/unit/{unit_id}/cards`

Get referenced unit's access cards (see `POST /management/unit/{id}/cards/attach`
and `POST /management/unit/{id}/cards/detach` endpoints) that the authorized client has access to.

#### Access: `user` or `supervisor` having `unit.view` permission for a unit and `card.view` permission for access cards (`card.view` permission is not required for `user` who is the owner of access card)

#### Request params:

*Optional:*

* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `10`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `name`, `description`, `creation_date`
  , `change_date`, `user_id`, `user_name`, `account_id` and `account_name`; default value is `name`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `asc`).

#### Response: `application/json`

```json
{
  "content": [
    {
      "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
      "account": {
        "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
        "name": "Test account"
      },
      "user": {
        "id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c",
        "name": "john.doe@vaultgroup.co.za"
      },
      "name": "card-123456789",
      "description": "The test card",
      "creation_date": "2022-06-09T11:26:13.289095Z",
      "change_date": "2022-06-09T17:22:36.708048Z",
      "pin_assigned": false
    },
    {
      ...
    },
    {
      ...
    },
    {
      ...
    }
  ],
  "page": 0,
  "page_size": 10,
  "total_pages": 3,
  "total_elements": 30
}
```

## POST `/management/card`

Create new access card.

#### Access: `supervisor` having `card.create` permission

#### Request body: `application/json`

```json
{
  "code": "123456789",
  "name": "card-123456789",
  "description": "Test access card"
}
```

*Required fields:*

* `code` a unique code of an access card (5-100 characters);
* `name` a non-unique human-readable name (up to 255 characters).

*Optional fields:*

* `description` an optional description of an access card (up to 10000 characters).

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
  "account": null,
  "user": null,
  "name": "card-123456789",
  "description": "The test card",
  "creation_date": "2022-06-09T11:26:13.289095Z",
  "change_date": "2022-06-09T11:26:13.289095Z",
  "pin_assigned": false
}
```

*Code is in use:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "code",
      "message": "Code is already in use"
    }
  ]
}
```

## PUT `/management/card/{id}`

Change existing access card by its identifier.

#### Access: `user` or `supervisor` having `card.edit` permission

#### Request body: `application/json`

```json
{
  "name": "my-super-card",
  "description": "Super card of mine"
}
```

*Required fields:*

* `name` a non-unique human-readable name (up to 255 characters).

*Optional fields:*

* `description` an optional description of an access card (up to 10000 characters).

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
  "account": null,
  "user": null,
  "name": "my-super-card",
  "description": "Super card of mine",
  "creation_date": "2022-06-09T11:26:13.289095Z",
  "change_date": "2022-06-09T11:26:13.289095Z",
  "pin_assigned": false
}
```

## POST `/management/card/{id}/assign`

Assign an access card to account and/or user/supervisor (set an ownership).
This endpoint can be used in four possible ways:

* `supervisor` assigns an access card to a certain account so the users of that account will manage the rest;
* `supervisor` assigns an access card to a certain account **and** a certain user within that account;
* `supervisor` assigns an access card to a certain account **and** a certain supervisor;
* `user` assigns an access card (that already belongs to their account) to some user (also can assign to themselves).

#### Access: `user` or `supervisor` having `card.edit` permission to access a card and `card.assign` permission to access account and/or user/supervisor (both old and new)

#### Request params:

* `{id}` an identifier of an access card to assign to account and/or user.

#### Request body: `application/json`

```json
{
  "account_id": "0697b16e-ddcc-4711-b072-906fdcc05270",
  "user_id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c"
}
```

*Fields:*

* `account_id` an identifier of an account to assign access card to (ignored for `user`, only applicable
  for `supervisor`);
* `user_id` an identifier of a user to assign access card to (required for `user`, optional for `supervisor`).

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "user": {
    "id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c",
    "name": "john.doe@vaultgroup.co.za"
  },
  "name": "card-123456789",
  "description": "The test card",
  "creation_date": "2022-06-09T11:26:13.289095Z",
  "change_date": "2022-06-09T17:22:36.708048Z",
  "pin_assigned": false
}
```

*Specified user does not belong to card's account:*

An access card cannot be assigned to a `supervisor` or a `user` of an account different from the specified one.

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "user_id",
      "message": "You cannot assign a card from one account to a user who belongs to a different one"
    }
  ]
}
```

## POST `/management/card/{id}/revoke`

Revoke an access card from user/supervisor or account it was assigned to.

* If the endpoint is being called by `supervisor` then the card is revoked from its owner (a user) **and** from account.
  In that case the card will also be detached from all the units that it was attached to (
  see `POST /management/unit/{id}/cards/attach` and `POST /management/unit/{id}/cards/detach` endpoints).
* If the endpoint is being called by `user` then the card is revoked from its owner (a user) **but** stays assigned to
  an account. In that case all the associations with units are kept untouched.

#### Access: `user` or `supervisor` having `card.edit` permission to access a card and `card.assign` permission to access account and/or user (depends on whether a card is assigned to a user or just to an account)

#### Request params:

* `{id}` an identifier of an access card to revoke from account and/or user.

#### Request body: `none`

#### Response

*Success (user):*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "user": null,
  "name": "card-123456789",
  "description": "The test card",
  "creation_date": "2022-06-09T11:26:13.289095Z",
  "change_date": "2022-06-09T17:35:12.165914Z",
  "pin_assigned": false
}
```

*Success (supervisor):*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
  "account": null,
  "user": null,
  "name": "card-123456789",
  "description": "The test card",
  "creation_date": "2022-06-09T11:26:13.289095Z",
  "change_date": "2022-06-09T17:35:12.165914Z",
  "pin_assigned": false
}
```

## POST `/management/card/{id}/pin`

Set a digital PIN (personal identification number) to an access card. The owner of the card is the only one who is
authorized to set a PIN. PIN is always reset to `null` when it's revoked or re-assigned.

#### Access: `user` or `supervisor` who is an owner of the access card

#### Request params:

* `{id}` an identifier of an access card to set PIN for.

#### Request body: `application/json`

```json
{
  "pin": "12345"
}
```

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "710d17f1-9dd5-4e5c-ac1c-ff4e51ea3aca",
  "account": {
    "id": "0697b16e-ddcc-4711-b072-906fdcc05270",
    "name": "Test account"
  },
  "user": {
    "id": "32af44d2-ae7d-44e8-9eb4-38580c3da77c",
    "name": "john.doe@vaultgroup.co.za"
  },
  "name": "card-123456789",
  "description": "The test card",
  "creation_date": "2022-06-09T11:26:13.289095Z",
  "change_date": "2022-06-09T17:22:36.708048Z",
  "pin_assigned": false
}
```

*Invalid PIN value:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "PIN must be a 5-10 digits number"
    }
  ]
}
```

## DELETE `/management/card/{id}`

Delete access card by its identifier.

#### Access: `supervisor` having `card.delete` permission

#### Request params:

* `{id}` an identifier of an access card to delete.

#### Response

*Success:*

* Response code: `204 No Content`
* Response body: `none`

*Card is assigned:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Cannot delete card. You have to revoke it first"
    }
  ]
}
```

## GET `/management/upgrades`

#### Access: `supervisor`

#### Request params:

*Optional:*

* `page` a 0-based number of a page to fetch;
* `size` a number of elements per page (by-default it's `10`);
* `sort` a sorting criterion (field) to use (supported values are: `id`, `name`, `description`, `registration_state`
  , `product_name`, `product_dimensions`, `product_description`, `creation_date`, `change_date`, `account_id`
  , `account_name`, `site_id` and `site_name`; default value is `name`);
* `order` a sorting direction (supported values are: `asc` and `desc`; default value is `asc`).

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "content": [
    {
      "id": "5859018b-4d27-4055-9355-65cfedf1801e",
      "type": "general",
      "name": "Update name",
      "description": null,
      "version_info": {
        "version": "1",
        "file": "BP434/BP216/rpi3-jessie/1/upgrade.tar.gz",
        "md5": "bBQbXJzRbaFP7Z0rkhUjxA=="
      },
      "hwos": "rpi3-jessie",
      "active": false,
      "unit_names": null,
      "reference": null,
      "tags": null,
      "creation_date": "2023-02-24T11:34:43.625743Z",
      "products": [
        "BP434",
        "BP216"
      ]
    },
    {
      "id": "9e80c2b4-c80f-4cf2-82f2-c80e53196130",
      "type": "general",
      "name": "2",
      "description": null,
      "version_info": {
        "version": "1",
        "file": "CNC419/CNC211/DNS427/rpi4-jessie/1/upgrade.tar.gz",
        "md5": "bBQbXJzRbaFP7Z0rkhUjxA=="
      },
      "hwos": "rpi4-jessie",
      "active": false,
      "unit_names": null,
      "reference": null,
      "tags": null,
      "creation_date": "2023-02-24T11:35:03.404262Z",
      "products": [
        "DNS427",
        "CNC211",
        "CNC419"
      ]
    },
    {
      "id": "5d39ee22-f571-4ef9-ab24-3aa195c9405e",
      "type": "general",
      "name": "Upgrade name 3",
      "description": null,
      "version_info": {
        "version": "1",
        "file": "LNS415/LNS207/rpi2-custom/1/upgrade.tar.gz",
        "md5": "bBQbXJzRbaFP7Z0rkhUjxA=="
      },
      "hwos": "rpi2-custom",
      "active": false,
      "unit_names": null,
      "reference": null,
      "tags": null,
      "creation_date": "2023-02-24T11:35:37.263132Z",
      "products": [
        "LNS207",
        "LNS415"
      ]
    }
  ],
  "page": 0,
  "page_size": 10,
  "total_pages": 1,
  "total_elements": 3
}
```

*Invalid credentials:*

* Response code: `401 Unauthorized`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid credentials"
    }
  ]
}
```

## GET `/management/upgrade`

#### Access: `unit`, no permissions required

#### Request body:

```json
{
  "file_name": "upgrade.tar.gz",
  "hwos": "rpi3-jessie",
  "version": "1",
  "tags": {
    "param1": "value1",
    "param2": "value2",
    "...": "..."
  }
}
```

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `binary file`

## GET `/management/upgrade/info?hwos={hwosType}`

#### Access: `unit`, no permissions required

#### Request params:

* `{hwosType}` Hardware and Operating System.

#### Request body: `application/json`

```json
{
  "tag1": "tag value",
  "tag2": "tag value",
  ...
}
```

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "version": "1",
  "file_name": "upgrade.tar.gz",
  "md5": "bBQbXJzRbaFP7Z0rkhUjxA=="
}
```

*No upgrade request code:*

* Response code: `204 No Content`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "No upgrade found"
    }
  ]
}
```

## POST `/management/upgrade`

Create a new upgrade.

#### Access: `supervisor` having `system.management` permission

#### Request body: `multipart/form-data`

* `upgrade_info` json that contains information about upgrade.

```json
{
  "name": string,
  "description": string,
  "type": string,
  "version": string,
  "hwos": string,
  "unit_names": string,
  "reference": string,
  "tags": object
}
```

* `{file}` multipart file.

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "3219ff88-91fd-4380-ba33-7987b210d426",
  "name": "Upgrade name",
  "description": "Upgrade description",
  "type": "general",
  "version_info": {
    "version": "1.0.0-test-upgrade",
    "file": "upgrade.tar.gz",
    "md5": "bBQbXJzRbaFP7Z0rkhUjxA=="
  },
  "hwos": "rpi3-bionic",
  "active": true,
  "unit_names": null,
  "reference": null,
  "tags": null,
  "creation_date": "2022-08-02T15:23:26.920512Z"
}
```

## PUT `/management/upgrade/{id}`

Change existing upgrade by its identifier.

#### Access: `supervisor` having `system.management` permission

#### Request params

*Required:*

* `{id}` an identifier of an upgrade.

#### Request body: `application/json`

```json
{
  "active": true,
  "description": "Upgrade description",
  "hwos": "rpi3-jessie"
}
```

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "id": "3219ff88-91fd-4380-ba33-7987b210d426",
  "name": "Upgrade name",
  "description": "Upgrade description",
  "type": "general",
  "version_info": {
    "version": "1.0.0-test-upgrade",
    "file": "upgrade.tar.gz",
    "md5": "bBQbXJzRbaFP7Z0rkhUjxA=="
  },
  "hwos": "rpi3-bionic",
  "active": true,
  "unit_names": null,
  "reference": null,
  "tags": null,
  "creation_date": "2022-08-02T15:23:26.920512Z"
}
```

## DELETE `/management/upgrade/{id}`

Delete upgrade by its identifier.

#### Access: `supervisor` having `system.management` permission

#### Request params:

* `{id}` an identifier of an upgrade to delete.

#### Response

*Success:*

* Response code: `204 No Content`
* Response body: `none`

*There are dependencies:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "There are dependent unit upgrades, you have to delete them first"
    }
  ]
}
```

*Upgrade not found:*

* Response code: `404 Not Found`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid upgrade id: 3219ff88-91fd-4380-ba33-7987b210d425"
    }
  ]
}
```

# Settings endpoints

## GET `/settings/unit`

Get current unit's custom settings. If `since` request parameter (date) is specified then the response will be `HTTP 204`
(with empty body) unless there are some changes to custom settings since `since` date.

#### Access: `unit`, no permissions required

#### Request params:

*Optional:*

* `since` if specified then the response will be `HTTP 204` unless last change date of the custom settings is greater
  than `since` value.

#### Response

*Settings changed or `since` is not provided:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "unit_id": "1213321e-abb9-480a-9de3-d2228e51ab89",
  "account_id": "18378836-b40a-4f71-b6de-5cd4fc868b53",
  "content": {
    "test": 777
  },
  "creation_date": "2023-06-02T19:41:06.719Z",
  "change_date": "2023-06-02T19:59:29.637Z",
  "unit_access_date": "2023-06-02T20:01:38.995Z"
}
```

*Settings are not set (don't exist)*

* Response code: `204 No Content`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "No settings for unit: 1213321e-abb9-480a-9de3-d2228e51ab89"
    }
  ]
}
```

*Settings didn't change since `since` date*

* Response code: `204 No Content`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "No Content"
    }
  ]
}
```

## GET `/settings/unit/{unit_id}`

Get referenced unit's custom settings.

#### Access: `user` or `supervisor` having `unit.view` permission

#### Response

*Success*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "unit_id": "1213321e-abb9-480a-9de3-d2228e51ab89",
  "account_id": "18378836-b40a-4f71-b6de-5cd4fc868b53",
  "content": {
    "test": 777
  },
  "creation_date": "2023-06-02T19:41:06.719Z",
  "change_date": "2023-06-02T19:59:29.637Z",
  "unit_access_date": "2023-06-20T12:48:42.036Z"
}
```

*Settings are not set*

* Response code: `204 No Content`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "No settings for unit: 1213321e-abb9-480a-9de3-d2228e51ab89"
    }
  ]
}
```

*Unit not found:*

* Response code: `404 Not Found`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid unit id: 1213321e-abb9-480a-9de3-d2228e51ab89"
    }
  ]
}
```

## PUT `/settings/unit`

Set current unit's custom settings (create or modify).

#### Access: `unit`, no permissions required

#### Request body: `application/json`

```json
{
  "foo": 1,
  "bar": 2,
  "baz": 3
}
```

The only requirement here: it must be a valid JSON document.

#### Response

*Success*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "unit_id": "1213321e-abb9-480a-9de3-d2228e51ab89",
  "account_id": "18378836-b40a-4f71-b6de-5cd4fc868b53",
  "content": {
    "foo": 1,
    "bar": 2,
    "baz": 3
  },
  "creation_date": "2023-06-02T19:41:06.719Z",
  "change_date": "2023-06-20T13:21:49.931304891Z",
  "unit_access_date": "2023-06-20T12:48:42.036Z"
}
```

*Settings object is too big*

* Response code: `413 Payload Too Large`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Payload Too Large, must be <= 102400 byte(s)"
    }
  ]
}
```

## PUT `/settings/unit/{unit_id}`

Set referenced unit's custom settings (create or modify).

#### Access: `user` or `supervisor` having `unit.edit` permission

#### Request params:

* `{unit_id}` an identifier of a unit to set settings for.

#### Request body: `application/json`

```json
{
  "foo": 1,
  "bar": 2,
  "baz": 3
}
```

The only requirement here: it must be a valid JSON document.

#### Response

*Success*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "unit_id": "1213321e-abb9-480a-9de3-d2228e51ab89",
  "account_id": "18378836-b40a-4f71-b6de-5cd4fc868b53",
  "content": {
    "foo": 1,
    "bar": 2,
    "baz": 3
  },
  "creation_date": "2023-06-02T19:41:06.719Z",
  "change_date": "2023-06-20T13:21:49.931304891Z",
  "unit_access_date": "2023-06-20T12:48:42.036Z"
}
```

*Settings object is too big*

* Response code: `413 Payload Too Large`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Payload Too Large, must be <= 102400 byte(s)"
    }
  ]
}
```

# Authentication endpoints

## POST `/authentication/unit/sign-up`

#### Access: `anonymous`

#### Request body: `application/json`

```json
{
  "registration_code": "30647151"
}
```

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "username": "unit123",
  "password": "2IUWxivRrtHppaGan1763717qbaDNugG",
  "dimensions": "1-1-2-2"
}
```

*Invalid registration request code:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "registration_code",
      "message": "Invalid registration request code"
    }
  ]
}
```

*Registration request code has expired:*

* Response code: `400 Bad Request`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "field": "registration_code",
      "message": "Registration request code has expired"
    }
  ]
}
```

## POST `/authentication/unit/sign-in`

#### Access: `anonymous`

#### Request body: `application/json`

```json
{
  "username": "unit123",
  "password": "2IUWxivRrtHppaGan1763717qbaDNugG"
}
```

#### Response

*Success:*

* Response code: `200 OK`
* Response body: `application/json`

```json
{
  "token": "eyJraWQiOiI0NmE2MzllNS1m..."
}
```

*Invalid credentials:*

* Response code: `401 Unauthorized`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Invalid credentials"
    }
  ]
}
```

*Unit is not in proper state:*

* Response code: `401 Unauthorized`
* Response body: `application/json`

```json
{
  "errors": [
    {
      "message": "Unit is not registered"
    }
  ]
}
```

## POST `/authentication/messaging/sign-in`

Get access credentials and other related information required to connect to a websocket server and receive instant notifications (dashboard events).

The given list of topics may contain exact topic names or topic patterns that the current user can subscribe to.
E.g. if `unit/#` is on the list then the user can subscribe to `unit/#` or any other topic that matches the pattern
(e.g. `unit/0697b16e-ddcc-4711-b072-906fdcc05270/a77392d8-5241-44a2-84a0-c12c76a6b532/#`).

#### Access: `user` or `supervisor` having `unit.logs.view` permission

#### Request body: `none`

#### Response: `application/json`

```json
{
  "endpoint": "wss://ws-saas.vaultgroup-cloud.com:8084/mqtt",
  "username": "john.doe@vaultgroup.co.za",
  "password": "eyJraWQiOiI0NmE2MzllNS1m...",
  "topics": [
    "unit/#",
    "unit:s/#"
  ]
}
```

[security]: security.md
[dashboard-rules]: dashboard-rules.md
