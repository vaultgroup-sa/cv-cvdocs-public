# Security

In order to access most of API endpoints you have to be an authorized **client**.

There are three types of **clients**: **units**, **accounts users** (or simply **users**) and **supervisors**.

**Units** are all the same so what they can or cannot do is predefined by API design. The entire permission system does only affect **account users** and **supervisors**.

**Account user** is basically the same thing as **supervisor**. The only difference is that **account user** belongs to a certain account and can only access resources (entities) within that account.

### Permissions

In order to define what given **client** can or cannot do we use **permission tokens** and **permission targets**.

**Permission token** represents an action that **client** can perform via API call. The scope of entities that can be **targets** of that action is represented by list of URN (uniform resource name) strings.

### Permission targets

URN starts with `urn:` prefix followed by entity type name (`account`, `site`, `unit` or `user`), `/` and entity's UUID.

Here are few examples:
- `urn:account/2438779b-3877-4992-9e0a-940fa4d22cc6`
- `urn:site/b5359e8f-ac93-4b58-9622-5fb941515b46`
- `urn:*`

A special URN is `urn:*` which stands for unlimited scope. Keep in mind that an **account user** cannot access anything outside their account so `urn:*` refers to everything as long as the target belongs to user's account. 

The URN defines the scope, not just one entity to have access to. Here's the hierarchy of entities:
- site always belongs to an account;
- unit always belongs to an account;
- unit as long as it is registered/installed also belongs to a site;
- account user always belongs to an account.

So in order to permit someone to read dashboard events of all units within certain site you don't need to refer to every single unit like this:
```json
{
  "tokens": ["unit.audit"],
  "target_urns": [
    "urn:unit/1e1bb59f-ef6e-446e-93e9-cb0f461faffc",
    "urn:unit/d17aa7f7-60b6-4338-97b2-225c93bd0611",
    "urn:unit/dca0a346-bb64-4b4d-b850-ec98edd84c9c",
    "urn:unit/7fbb3e8c-ac62-49d7-9206-db8f5203fa75",
    "urn:unit/bdd8c42c-7edf-4be1-b8c4-4a6e37227411"
  ]
}
```

It'd be sufficient to refer to the whole site instead:
```json
{
  "tokens": ["unit.audit"],
  "target_urns": ["urn:site/b5359e8f-ac93-4b58-9622-5fb941515b46"]
}
```

### Available permissions

| Token                   | Description                                                                                                                                                                                                                                                                                 | Supported targets                                                              |
|-------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------|
| `account.view`          | Permission to read basic account information. Defines a scope of visible accounts that can be seen via API endpoints.                                                                                                                                                                       | `urn:*`, `urn:account/{id}`                                                    |
| `account.create`        | Permission to create new accounts. Can only be granted to `supervisors`.                                                                                                                                                                                                                    | Does not support any targets.                                                  |
| `account.edit`          | Permission to edit basic account information.                                                                                                                                                                                                                                               | `urn:*`, `urn:account/{id}`                                                    |
| `account.delete`        | Permission to delete existing account.                                                                                                                                                                                                                                                      | `urn:*`, `urn:account/{id}`                                                    |
| `site.view`             | Permission to read basic site information. Defines a scope of visible sites that can be seen via API endpoints.                                                                                                                                                                             | `urn:*`, `urn:account/{id}`, `urn:site/{id}`                                   |
| `site.create`           | Permission to create new sites.                                                                                                                                                                                                                                                             | `urn:*`, `urn:account/{id}`                                                    |
| `site.edit`             | Permission to edit basic site information.                                                                                                                                                                                                                                                  | `urn:*`, `urn:account/{id}`, `urn:site/{id}`                                   |
| `site.delete`           | Permission to delete existing sites.                                                                                                                                                                                                                                                        | `urn:*`, `urn:account/{id}`, `urn:site/{id}`                                   |
| `unit.view`             | Permission to read basic unit information and its settings. Defines a scope of visible units that can be seen via API endpoints                                                                                                                                                             | `urn:*`, `urn:account/{id}`, `urn:site/{id}`, `urn:unit/{id}`                  |
| `unit.audit`            | Permission to read dashboard events (and receive MQTT messages generated by dashboard rules), to read unit diagnostic reports and manage their status, to manage unit servicing records (only `supervisors` can manage unit service mode).                                                  | `urn:*`, `urn:account/{id}`, `urn:site/{id}`, `urn:unit/{id}`                  |
| `unit.create`           | Permission to create new units.                                                                                                                                                                                                                                                             | `urn:*`, `urn:account/{id}`                                                    |
| `unit.edit`             | Permission to edit basic unit information and its settings.                                                                                                                                                                                                                                 | `urn:*`, `urn:account/{id}`, `urn:site/{id}`, `urn:unit/{id}`                  |
| `unit.mqtt`             | Permission to exchange MQTT messages with a unit.                                                                                                                                                                                                                                           | `urn:*`, `urn:account/{id}`, `urn:site/{id}`, `urn:unit/{id}`                  |
| `unit.vpn`              | Permission to use VPN and enable/disable it for a unit.                                                                                                                                                                                                                                     | `urn:*`, `urn:account/{id}`, `urn:site/{id}`, `urn:unit/{id}`                  |
| `unit.registration`     | Permission to manage unit's registration (change registration state, schedule or reset registration, view unit registration request if registration is scheduled). **Attention**: in order to schedule a unit registration at certain site you need a `site.view` permission for that site. | `urn:*`, `urn:account/{id}`, `urn:site/{id}`, `urn:unit/{id}`                  |
| `unit.delete`           | Permission to delete existing units.                                                                                                                                                                                                                                                        | `urn:*`, `urn:account/{id}`, `urn:site/{id}`, `urn:unit/{id}`                  |
| `user.view`             | Permission to read basic user information. Defines a scope of visible accounts that can be seen via API endpoints.                                                                                                                                                                          | `urn:*`, `urn:account/{id}`, `urn:user/{id}`                                   |
| `user.create`           | Permission to create new users.                                                                                                                                                                                                                                                             | `urn:*`, `urn:account/{id}`                                                    |
| `user.edit`             | Permission to edit basic user information.                                                                                                                                                                                                                                                  | `urn:*`, `urn:account/{id}`, `urn:user/{id}`                                   |
| `user.permissions.edit` | Permission to view and edit other user's permissions.                                                                                                                                                                                                                                       | `urn:*`, `urn:account/{id}`, `urn:user/{id}`                                   |
| `user.delete`           | Permission to delete existing users.                                                                                                                                                                                                                                                        | `urn:*`, `urn:account/{id}`, `urn:user/{id}`                                   |
| `card.view`             | Permission to read basic access cards information.                                                                                                                                                                                                                                          | `urn:*`, `urn:account/{id}`, `urn:user/{id}`, `urn:card/{id}`                  |
| `card.create`           | Permission to create new access cards.                                                                                                                                                                                                                                                      | Does not support any targets.                                                  |
| `card.edit`             | Permission to edit basic information and bindings (assign to or revoke from accounts/users, attach to or detach from units) of access cards.                                                                                                                                                | `urn:*`, `urn:account/{id}`, `urn:user/{id}`, `urn:card/{id}`                  |
| `card.delete`           | Permission to delete access cards. Can only be granted to `supervisors`.                                                                                                                                                                                                                    | `urn:*`, `urn:account/{id}`, `urn:card/{id}`                                   |
| `card.assign`           | Permission to assign and revoke access cards to/from accounts and/or users. Also permits to attach/detach access cards to/from units. Only `supervisors` can change access card's account.                                                                                                  | `urn:*`, `urn:account/{id}`, `urn:site/{id}`, `urn:user/{id}`, `urn:unit/{id}` |
| `sms.send`              | Permission to send SMS on behalf of the platform. Can be granted to `users` and `supervisors`. The `units` can send SMSes without permissions.                                                                                                                                              | Does not support any targets.                                                  |
| `system.management`     | A general permission to manage certain system settings or entities. Allows an access to manage the models and firmware categories. Can only be granted to `supervisors`.                                                                                                                    | Does not support any targets.                                                  |

All the entities are completely separate in terms of permissions effect. So in order to have a read access to unit you
**don't** need to have a read access to the site or account of that unit.
