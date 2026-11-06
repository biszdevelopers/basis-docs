# DevConnect FAQ

## What is DevConnect?

DevConnect is the user-facing name for the basis-auth OpenID Connect service.
It lets registered applications use a Microsoft Entra sign-in and receive
tokens for their own resource APIs.

## What happens when I sign in?

The app sends you to DevConnect, which signs you in through Microsoft Entra.
The app then receives an authorization code and exchanges it for tokens.

```md
![Sign In Screen](../images/1.png)
```

You may see a Microsoft sign-in prompt:

```md
![Microsoft Sign In](../images/3.png)
```

Review the application's requested access on the consent screen. Choose
**Allow** to continue or **Deny** to cancel.

```md
![Consent Screen](../images/2.png)
```

Microsoft handles your Microsoft password and any Microsoft MFA prompt.
DevConnect does not ask you to create a separate password.

## What information does an app receive?

Each app requests registered scopes for one resource audience. Its access
token can contain permissions granted for that app. Applications should use
only the scopes and permissions needed for their features.

The provider retrieves basic account information and a profile picture from
Microsoft. App-specific resource scopes are not forwarded to Microsoft as
Microsoft Graph permissions.

## Does signing out of one app sign me out everywhere?

Each app controls its own application session. An app may also sign out of
the DevConnect SSO session; that makes the next app ask DevConnect to sign in
again. Signing out of DevConnect does not sign you out of Microsoft, so
Microsoft may not ask for your password again.

## Why can I not sign in to an app?

The account must be allowed by that app's configured organization and email
domain, and it must pass the app's optional email allow-list or block-list.
Choose the intended Microsoft account. If access still fails, contact the
application owner or the organization's support team.

The default database migrations map `basischina.com` and `basis-global.com`
to the default organization. Operators can configure additional suffixes and
organizations for other deployments.

## Where should I report a problem?

Contact the owner or support contact for the application you are trying to use.
For an account or provider issue, contact the team that operates DevConnect.

## What should I do if Microsoft sign-in shows an error?

Check that you selected the correct account and follow any Microsoft prompt
for password reset, MFA, or organization policy. If the problem continues,
send the displayed error code to your application owner or DevConnect operator.
