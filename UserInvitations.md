# User Invitations

Users are attached to `Org`s; this is done by creating a linking `OrgUser` object between the `User` and the `Org`.

When an existing platform user (having the appropriate permissions) invites a collaborator, they do this by providing the email address of the intended collaborator. We use this email address to look up a `User` object.

If a `User` is found, then we simply create the `OrgUser` and send an email notifying them that they have been added to another `Org`.

If no `User` exists, we create an `Invitation` object tracking

- the inviter
- the invitee
- the role being granted
- the timestamp

(The `Org` being invited to is implied via the inviter).

We then generate a unique invitation code which we attach to a URL, embed into an email and send to the invitee.

When they click the link in the email they are brought to a verification page which verifies the user by looking up the `Invitation` by the invitation code.

This verification process creates the `User` and the `OrgUser` attached to the `Org` of the inviter.
