# V2 Usage Instructions


These instructions presume you have already created your
[Adobe.IO](https://www.adobe.io/) integration, as described
in the [home page](index.html) of this documentation.

# Getting a Connection

All UMAPI access is predicated on the creation of an authenticated,
authorized connection to the UMAPI server.  This access always
happens in the context of a particular integration (as created
on adobe.io).  So it requires the following details about
your integration:

1. Organization ID
2. Tech Account ID
3. IMS Hostname
4. IMS Token Exchange Endpoint (aka JWT Endpoint)
5. API Key
6. Client Secret
7. Private Key File (unencrypted form)

Of these, the IMS Hostname and the IMS Token Exchange Endpoint
are standard across almost all integrations, so they are
built into the library as defaults and aren't typically needed.
The Tech Account ID, API Key, and Client Secret are sensitive,
as is the Private Key File, so a best practice is to keep
these values in files separate from your application.  The
Organization ID is not as sensitive, but it's best to keep
it with the others since it's also needed for authentication.

For example, suppose `config.yaml` is a YAML file
whose content contains
the sensitive data (elided here for security):

```yaml
org_id: '620049..............101@AdobeOrg'
tech_acct_id: '78E9928............A495DE3@techacct.adobe.com'
api_key: '265434.............d740ac'
client_secret: 'cc6.....-....-47b9-....-......ff3725'
private_key_file: '/path/to/my.secret.key.pem'
```

and suppose `my.secret.key.pem` contains an unencrypted
private key
like this (again, elided for security):

```
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAxBc5BFNUP9hdGHSuOzfxoyL2qq2qcqpSexLsefQS9fDZZjCP
...
fIOe8cq8F5Vcw6l5NwmW+Lw44hJxKAVRg+j79x6C6+zLblRhm+dHBw==
-----END RSA PRIVATE KEY-----
```

Given these two files, you can establish an authenticated,
authorized connection with code such as the following:

```python
import yaml # PyYAML from PyPI provides this module

with open(config_file_name, "r") as f:
    config = yaml.load(f)
conn = umapi_client.Connection(org_id=config["org_id"],
                               auth_dict=config)
```

The constructor of the Connection object will do all the
work of contacting the token exchange endpoint and using
your credentials to obtain an access token, and it will
remember that access token for use with all your UMAPI
operations.  It will be called `conn` in the examples
that follow.

(If you want the details of how access token exchange
is done by the constructor, see the code or the
[v1 usage docs](usage-instructions-v1.html).)

# Querying for Users and Groups

Queries for users and groups are implemented by
classes which allow iterating the results.  These
iterators pull the results from the server in
batches of 200 or so, and cache the results locally.
You can access the full list of results with
the `all_results` method, and force the query
to be reloaded and run from the beginning again
with the `reload` method.

Each fetched user or group is represented as
a Python dictionary of its attributes.

## Get a List of Users

The following code enumerates the first 5 users, printing the email of each.
This will only have fetched the first page of results.  Then, after the loop,
the `all_results` call will force the fetch of all remaining pages so the
list of all results can be constructed.  ()Once the `all_results` call has been made,
you cannot enumerate again without first calling `reload`.)

```python
users = umapi_client.QueryUsers(conn)
# print first 5 users
for i, user in enumerate(users):
    if i == 5: break
    print("User %d email: %s" % (i, user["email"]))
# get a count of all users (finishes the iteration)
user_count = len(users.all_results())
```

## Get a List of Groups

This list of groups will contain both user groups and product license
configuration groups.

```python
groups = umapi_client.QueryGroups(conn)
# print all the group details
for group in groups:
    print(group)
# after an hour, see if anything has changed
time.sleep(3600)
groups.reload()
for group in groups:
    print(group)
```

# Performing Operations on Users

User operations in the UMAPI are performed in three steps:

1. You specify the user to be operated on.
2. You specify the operations to be performed on the user.
3. You submit the user and operations to the UMAPI server.

The combined specification of the user identity and the operations to be performed
is called an _action_ in the UMAPI documentation, while the individual operations are called
_commands_.  If you read the documentation carefully, you will see that there
are limits to how many actions can be submitted to the UMAPI
service in a single call, how many commands there can be in a single action, and
how many calls can be submitted in a given period of time.  However,
the `umapi_client` implementation has been design to insulate
your application from these limits
by  
packing as many commands as allowed into each action,
batching up as many actions as possible into a call, 
and spacing calls out when required to by the server.  Thus, from an
application perspective, you can simply follow the three steps above for each
user and not worry about the mechanics of server communication limits.

## Step 1: Specify the User

To operate on a user, you first create a `UserAction` object that
specifies the user's identity type, domain, and unique ID in the domain.

In most cases,
the user's email ID will be his unique ID and will itself contain his domain,
as in these examples:

```python
from umapi_client import IdentityTypes
user1 = UserAction(id_type=IdentityTypes.adobeID, email="user@isp.net")
user2 = UserAction(id_type=IdentityTypes.enterpriseID, email="user@company.com")
```

But when Federated ID is being used, and a non-email username is being
used to identify users across the SAML connection, both the username
and the domain must be specified separately, as in these examples:

```python
user3 = UserAction(id_type=IdentityTypes.federatedID,
                   username="user347", domain="division.conglomerate.com")
user4 = UserAction(id_type=IdentityTypes.federatedID,
                   username="user348", domain="division.conglomerate.com",
                   email="john.r.user@conglomerate.com")
```

Note that, as in the last example, it's OK to specify the email when
creating a user object even if the email is not the unique ID or
doesn't use the same domain.  If
you later perform an operations on a user which requires the email
(such as user creation on the Adobe side), the email will be remembered
and supplied from the UserAction object.

## Step 2: Specify the Operations

Once you have a `UserAction` object for a user, you can specify
operations (called _commands_) to perform on that user.  For
example, to create a new user on the Adobe side, for the users
that were specified in the last section, we could do:

```python
user1.create()
user2.create(first_name="Geoffrey", last_name="Giraffe")
user3.create(email="jane.doe@conglomerate.com", country="US")
user4.create(first_name="John", last_name="User", country="US")
```

When creating users, the email address is mandatory if not already specified
when creating the user action.  First and last name and country can be optionally
specified ()except for Adobe ID users),
and country _must_ be specified for Federated ID users.

If a user has already been created, but you want to update attributes,
you can use the `update` rather than the `create` command:

```python
user2.update(first_name="Jeff", country="AU")
user4.update(username="user0347")
```

You can also specify to create if necessary, but update if already created:

```python
from umapi_client import IfAlreadyExistsOptions
user4.create(first_name="John", last_name="User", country="US",
             on_conflict=IfAlreadyExistsOptions.updateIfAlreadyExists)
```

There are many other operations you can perform, such as adding and removing
users from user groups and product configuration groups.  Because each
operation specifier returns the user, it's easy to chain the together:

```python
user2.add_group(groups=["Photoshop", "Illustrator"]).remove_group(groups=["CC All Apps"])
```

The details of all the possible commands are specified in the code,
and more user documentation will be forthcoming.  In general, commands
are performed in the order they are specified, except for certain special
commands such as ```create``` which are always performed first regardless
of when they were specified.

## Step 3: Submit to the UMAPI server

Once you have specified all the desired operations on a given `UserAction`,
you can submit it to the server as follows (recall that `conn` is an authorized
connection to the UMAPI server, as created above):

```python
result = conn.execute_single(user1)
result = conn.execute_multiple([user2, user3, user4])
```

By default, `execute_single` queues the action for sending to the server
when a "full batch" (of 10) actions has been accumulated, but
`execute_multiple` forces a batch to be sent (including any
previously queued actions as well as the specified ones).  You can
override these defaults with the `immediate` argument, as in:

```python
result = conn.execute_single(user1, immediate=True)
result = conn.execute_multiple([user2, user3, user4], immediate=False)
```

The result of either execute operation is a tuple of three numbers
`(queued, sent, succeeded)` which tell you how many actions were
queued, how many were sent, and how many of those sent succeeded
without errors.  So, for example, in the following code:

```python
queued, _, _ = conn.execute_single(user1)
_, sent, succeeded = conn.execute_multiple(user2, user3, user4)
```

we would likely see `queued = 1`, `sent = 4`, and `succeeded = 4`.

If, for some reason, the succeeded number is not equal to the sent
number for any call, it means that not all of the actions were
executed successfully on the server-side: one or more of the commands
failed to execute.  In such cases, the server will have sent back
error information which the `umapi_client` implementation records
against the commands that failed, you can call the `execution_errors`
method on the user actions to get a list of the failed commands
and the server error information.  For example, if only three
of the four actions sent had succeeded, then we could execute
this code:

```python
actions = (user1, user2, user3, user4)
errors = [info for action in actions for info in action.execution_errors()]
```

Each entry in `errors` would then be a dictionary giving
the command that failed, the target user it failed on,
and server information about the reason for the failure.

