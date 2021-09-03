# Spam checker callbacks

Spam checker callbacks allow module developers to implement spam mitigation actions for
Synapse instances. Spam checker callbacks can be registered using the module API's
`register_spam_checker_callbacks` method.

The available spam checker callbacks are:

```python
async def check_event_for_spam(event: "synapse.events.EventBase") -> Union[bool, str]
```

Called when receiving an event from a client or via federation. The module can return
either a `bool` to indicate whether the event must be rejected because of spam, or a `str`
to indicate the event must be rejected because of spam and to give a rejection reason to
forward to clients.

```python
async def user_may_invite(inviter: str, invitee: str, room_id: str) -> bool
```

Called when processing an invitation. The module must return a `bool` indicating whether
the inviter can invite the invitee to the given room. Both inviter and invitee are
represented by their Matrix user ID (e.g. `@alice:example.com`).

```python
async def user_may_create_room(user: str) -> bool
```

Called when processing a room creation request. The module must return a `bool` indicating
whether the given user (represented by their Matrix user ID) is allowed to create a room.

```python
async def user_may_create_room_alias(user: str, room_alias: "synapse.types.RoomAlias") -> bool
```

Called when trying to associate an alias with an existing room. The module must return a
`bool` indicating whether the given user (represented by their Matrix user ID) is allowed
to set the given alias.

```python
async def user_may_publish_room(user: str, room_id: str) -> bool
```

Called when trying to publish a room to the homeserver's public rooms directory. The
module must return a `bool` indicating whether the given user (represented by their
Matrix user ID) is allowed to publish the given room.

```python
async def check_username_for_spam(user_profile: Dict[str, str]) -> bool
```

Called when computing search results in the user directory. The module must return a
`bool` indicating whether the given user profile can appear in search results. The profile
is represented as a dictionary with the following keys:

* `user_id`: The Matrix ID for this user.
* `display_name`: The user's display name.
* `avatar_url`: The `mxc://` URL to the user's avatar.

The module is given a copy of the original dictionary, so modifying it from within the
module cannot modify a user's profile when included in user directory search results.

```python
async def check_registration_for_spam(
    email_threepid: Optional[dict],
    username: Optional[str],
    request_info: Collection[Tuple[str, str]],
    auth_provider_id: Optional[str] = None,
) -> "synapse.spam_checker_api.RegistrationBehaviour"
```

Called when registering a new user. The module must return a `RegistrationBehaviour`
indicating whether the registration can go through or must be denied, or whether the user
may be allowed to register but will be shadow banned.

The arguments passed to this callback are:

* `email_threepid`: The email address used for registering, if any.
* `username`: The username the user would like to register. Can be `None`, meaning that
  Synapse will generate one later.
* `request_info`: A collection of tuples, which first item is a user agent, and which
  second item is an IP address. These user agents and IP addresses are the ones that were
  used during the registration process.
* `auth_provider_id`: The identifier of the SSO authentication provider, if any.

```python
async def check_media_file_for_spam(
    file_wrapper: "synapse.rest.media.v1.media_storage.ReadableFileWrapper",
    file_info: "synapse.rest.media.v1._base.FileInfo",
) -> bool
```

Called when storing a local or remote file. The module must return a boolean indicating
whether the given file can be stored in the homeserver's media store.