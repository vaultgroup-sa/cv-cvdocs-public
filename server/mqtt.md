# MQTT

As a part of SaaS platform the MQTT provides a channel for real time UI notifications and third party integration.

## Authorization

In order to connect to MQTT broker you require a SaaS user first. If you don't have one, ask VaultGroup for an invitation.

Once you're logged in on behalf of the SaaS user you can call an API endpoint (see `/authentication/messaging/sign-in` endpoint in [api]) to get the MQTT broker URL and your MQTT access credentials.

The password for MQTT broker authorization is a JWT which will expire after some time. So you'll have to request fresh credentials periodically to stay connected.

## Security

The SaaS platform provides a fine-grained control (see [security] for more) over user access to the data. An MQTT client
shares the same restrictions as the SaaS user.

The `topics` list (see `/authentication/messaging/sign-in` endpoint in [api]) is not fixed! Its contents reflect user's permissions. Thus, certain topics may or may not be available for subscription.

As shown below the MQTT topics have a hierarchical structure so as the access permissions. Here's an example

For example a user is only granted a view permission on all the units of two accounts (account_id = `f4f8426e-ac90-4ce5-9406-f78523c10f4f` and account_id = `c719fb58-c849-4484-a303-a2ac4809e230`).
In that case the user won't be allowed to subscribe to a topic pattern `entity/unit/+/+/+` which would've exposed the data of other accounts too.
Instead, the user will only be allowed to subscribe to `entity/unit/f4f8426e-ac90-4ce5-9406-f78523c10f4f/+/+` and `entity/unit/c719fb58-c849-4484-a303-a2ac4809e230/+/+` topic patterns.

That will be appropriately reflected in the `topics` list.

## Entity topic concept

All the topics described below have one thing in common: the messages are published to those topics with `retain` option set to `true` (see https://www.emqx.com/en/blog/mqtt5-features-retain-message for detailed explanation).

The idea is that every topic represents a single entity. The topic exists for as long as the entity does.
When a client subscribes to that topic (or to a topic pattern matching that topic) a **retained** message is immediately delivered to the client.
That message represents the last known state of the entity. When entity is deleted the message is deleted as well so as the topic.
When entity is updated a new **retained** message replaces the previous one.

So no matter if a client stays connected, connects for the first time or reconnects after access token expiration, the client will always receive the very last known state of the entity and nothing else.

## Account topics

Full access subscription pattern (any account): `entity/account/+`

### `entity/account/{account_id}`

Topic example: `entity/account/f4f8426e-ac90-4ce5-9406-f78523c10f4f`

```json
{
  "id": "f4f8426e-ac90-4ce5-9406-f78523c10f4f",
  "name": "VG Account",
  "description": "The VaultGroup Account",
  "creation_date": "2022-08-01T12:23:49.202604Z",
  "change_date": "2022-08-01T12:33:49.450056Z"
}
```

## Site topics

A site always belongs to some account.

Full access subscription pattern (any site of any account): `entity/site/+/+`

### `entity/site/{account_id}/{site_id}`

Topic example: `entity/site/f4f8426e-ac90-4ce5-9406-f78523c10f4f/48c77a46-fd7a-4802-a6a5-881bc1eab630`

```json
{
  "id": "48c77a46-fd7a-4802-a6a5-881bc1eab630",
  "account_id": "f4f8426e-ac90-4ce5-9406-f78523c10f4f",
  "name": "VG Office",
  "description": "The VaultGroup Office",
  "creation_date": "2025-09-11T09:34:46.400940Z",
  "change_date": "2025-09-11T09:34:46.400940Z"
}
```

## Unit topics

A unit always belongs to some account, but it may or may not be registered in some site.

Full access subscription pattern (any unit of any site of any account): `entity/unit/+/+/+`

### `entity/unit/{account_id}/{site_id}/{unit_id}`

Topic example: `entity/unit/f4f8426e-ac90-4ce5-9406-f78523c10f4f/48c77a46-fd7a-4802-a6a5-881bc1eab630/cfcf01b7-0f67-470b-a92c-f595ab30bff4`

```json
{
  "id": "cfcf01b7-0f67-470b-a92c-f595ab30bff4",
  "account_id": "f4f8426e-ac90-4ce5-9406-f78523c10f4f",
  "site_id": "48c77a46-fd7a-4802-a6a5-881bc1eab630",
  "name": "cv123",
  "description": "Unit #123",
  "registration_state": "REGISTERED",
  "model_id": "aaaeabe6-0936-41fc-b89a-fe2237ea52e8",
  "product_id": "dcef72b7-e850-4e07-bac2-0cbd0a6abe89",
  "firmware_category_id": "62fcec3e-c97b-11ed-afa1-0242ac120002",
  "hwos": "rpi3-focal",
  "tags": {"branch": "default","location": "default"},
  "is_upgrade_enabled": true,
  "upgrade_version": null,
  "upgrade_checksum": null,
  "upgrade_timestamp": null,
  "upgrade_override_version": null,
  "upgrade_override_checksum": null,
  "creation_date": "2023-05-02T07:08:09.295752Z",
  "change_date": "2025-08-28T08:37:30.673670Z"
}
```

### `entity/unit/{account_id}/-/{unit_id}`

Topic example: `entity/unit/5c620dbe-171a-4ab3-be3f-407b2b7c8eb8/-/be4a4828-35fb-4a44-a10e-32864b305c9b`

```json
{
  "id": "be4a4828-35fb-4a44-a10e-32864b305c9b",
  "account_id": "5c620dbe-171a-4ab3-be3f-407b2b7c8eb8",
  "site_id": null,
  "name": "cv234",
  "description": null,
  "registration_state": "UNREGISTERED",
  "model_id": "83355da1-7f51-4b1a-9a71-4e21c64fa10a",
  "product_id": "b3de4599-c8c0-403e-b52b-b14df3cd6175",
  "firmware_category_id": "55e1c57e-c97b-11ed-afa1-0242ac120002",
  "hwos": "rpi3-focal",
  "tags": {"branch": "default", "location": "default"},
  "is_upgrade_enabled": true,
  "upgrade_version": null,
  "upgrade_checksum": null,
  "upgrade_timestamp": null,
  "upgrade_override_version": null,
  "upgrade_override_checksum": null,
  "creation_date": "2025-10-06T15:22:06.157406Z",
  "change_date": "2025-10-06T15:22:06.157406Z"
}
```

## User topics

A user may or may not belong to an account. A user without account is also known as `supervisor`.

Full access subscription pattern (any user of any account or any supervisor): `entity/user/+/+`

### `entity/user/{account_id/{user_id}`

Topic example: `entity/user/f4f8426e-ac90-4ce5-9406-f78523c10f4f/52bcdd12-c992-43eb-a6b3-4587dc772036`

```json
{
  "id": "52bcdd12-c992-43eb-a6b3-4587dc772036",
  "account_id": "f4f8426e-ac90-4ce5-9406-f78523c10f4f",
  "username": "example.com",
  "full_name": "John Smith",
  "active": true,
  "creation_date": "2023-04-03T06:43:03.598855Z",
  "change_date": "2023-04-03T14:03:51.171928Z"
}
```

### `entity/user/-/{user_id}`

Topic example: `entity/user/-/099e507a-962c-49c5-a7ff-31031b2a7459`

```json
{
  "id": "099e507a-962c-49c5-a7ff-31031b2a7459",
  "account_id": null,
  "username": "johndoe@vaultgroup.co.za",
  "full_name": "John Doe",
  "active": true,
  "creation_date": "2024-06-20T07:34:18.640963Z",
  "change_date": "2024-06-20T07:46:37.016187Z"
}
```

## Unit model topics

Full access subscription pattern: `entity/unit_model/+`

### `entity/unit_model/{model_id}`

Topic example: `entity/unit_model/1cb9f2bb-f1f4-ee92-a98d-d7077d8ded0a`

```json
{
  "id": "1cb9f2bb-f1f4-ee92-a98d-d7077d8ded0a",
  "name": "FOH314",
  "dimensions": "4-5-5",
  "description": "Front Of House 3 x 14",
  "creation_date": "2022-05-24T14:56:34.341792Z",
  "change_date": "2022-05-24T14:56:34.341792Z"
}
```

## Unit product topics

Full access subscription pattern: `entity/unit_product/+`

### `entity/unit_product/{product_id}`

Topic example: `entity/unit_product/932b56ba-4a70-4f3b-b01f-c7f830c1eb1f`

```json
{
  "id": "932b56ba-4a70-4f3b-b01f-c7f830c1eb1f",
  "name": "CellVault",
  "description": null,
  "creation_date": "2025-08-18T07:42:17.065388Z",
  "change_date":"2025-08-18T07:42:17.065388Z"
}
```

[api]: api.md
[security]: security.md
