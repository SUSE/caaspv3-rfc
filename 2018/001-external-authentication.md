# External Authentication Integration

| Field  | Value      |
|:-------|:-----------|
| Status | Draft      |
| Date   | 2018-05-03 |

## Introduction

### Problem description

Many enterprises will have a pre-existing authentication service such as LDAP,
Active Directory, SAML 2.0, etc and wish to use it to handle authentication of
users for Kubernetes.

### Proposed change

* Allow operators to provide connection details for external authentication
  systems via Velum.
* Configure Dex with these connection details, enabling Kubernetes to
  authenticate, via Dex, against these external services.
* Support external LDAP, SAML 2.0 and OpenID Connect authentication
  services.

## Detailed RFC

Kubic/CaaSP currently uses Dex as an authentication intermediary for
Kubernetes, where Dex is backed by a Kubic managed LDAP server.

Dex supports many authentication "connectors" - LDAP, SAML 2.0, OpenID Connect,
as well as several less common connectors such as GitHub, LinkedIn, etc. Dex
additionally supports configuring multiple connectors at once, allowing users
to choose which service they need to authenticate against.

This RFC proposes allowing Kubic operators to configure arbitary additional
connectors within Kubic, pointing towards their own authentication backend.
Initially, this will be limited to Dex's LDAP, SAML 2.0 and OpenID Connect
connectors.

This RFC does not propose removing that managed LDAP server, though that may
be something we wish to pursue in the future.

### Proposed change

#### Velum Workflow Changes

We propose to leave the initial bootstrap workflow unchanged, and instead
allow additional authentication connectors to be configured post-bootstrap, in
a similar manner to how we allow containter registries to be configured
post-bootstap.

Operators should be able to login to Velum, navigate to the Settings tab, and
view, add or remove authentication connectors. Each authentication connector
type (LDAP, SAML 2.0, OpenID Connect) will have a different view for the view
and add pages, as Dex requires a different set of configuration inputs for
each.

It should be possible for operators to configure multiple connectors
simultaneously, including multiple connectors of the same type (e.g. 2x LDAP).

Once connectors have been configured, Velum will show an "Apply Changes" button,
again - similar to container registries page, and Salt will be called to push
out the updated configuration.

#### Salt Implementation

Velum will store the connector details in it's database, and will render the
values into the Pillar API endpoint for Salt to read.

Once in salt, most of the pieces are already in place for this. We'll need to
update the Dex ConfigMap template (https://github.com/kubic-project/salt/blob/292b025681915a6ad4d7378624385d231f2f21d4/salt/addons/dex/manifests/15-configmap.yaml#L21)
to render the new connector details, and then salt will apply the change to
Kubernetes, causing a roll out of the change.

#### CaaSP CLI Implementation

As we now support SAML 2.0 and OpenID Connect, the current `caasp-cli` workflow
will no longer be possible. CaaSP CLI must be updated to fully implement the
OIDC browser based workflow. When a user logs in with `caasp-cli`, they must be
presented with a URL to open in their browser, where they will login and copy
the generated token back to a waiting `caasp-cli` in order to complete the
authentication flow. The existing expectations that `caasp-cli` can provide the
username and password on behalf of the user to Dex are no longer valid.

### Dependencies

This change should be reasonably isolated, simply providing the UI to enter the
necessary data, and plumbing that through to Dex.

However, documentation and testing efforts will be needed to ensure this feature
is complete, and in working order.

### Concerns and Unresolved Questions

1. Dex will be restarted, in a rolling fashion, as the configmap is updated.
   This may introduce authentication issues during the change.
2. It's not known how Dex will handle an incorrectly configured connector,
   should Dex end up in a reboot-loop, we're likely to have an issue.
3. Not all connectors support passing through group membership information. This
   will need to checked for each connector we enable, and documented as such
   within the Velum UI.
4. Dex has a limited number of connectors. As an example, Dex does not support
   delegating authentication to OpenStack Keystone. Should this become a
   requirement, we'll need to build this support into Dex.
5. Dex will likely not survive as an active project forever - however, the
   interface we build against is OpenID Connect, and any similar project in this
   space may be dropped in to replace Dex should the need arise.

## Alternatives

Kubernetes itself supports many authentication mechanisms - X509 certs, static
credentials, webhook auth, proxy auth and OpenStack keystone auth without the 
use of Dex. However, implementing any of these choices either does not satisfy
the goal (e.g. static credentials), or would require significantly more effort
to implement and use with Velum (e.g. Webhook, Keystone, etc).

By delegating authentication to Dex, we can implement a single interface (OpenID
Connect to Dex), and have Dex worry about the details for further delgation into
the customer managed authentication service.

## Revision History:

| Date       | Comment       |
|:-----------|:--------------|
| 2018-05-03 | Initial Draft |
