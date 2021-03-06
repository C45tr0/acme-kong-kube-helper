# acme-kong-kube-helper
A kong-ingress-controller helper utility ~~needed short-term~~ no longer needed

[![Docker Pulls](https://img.shields.io/docker/pulls/ollystephens/acme-kong-kube-helper.svg)](https://hub.docker.com/r/ollystephens/acme-kong-kube-helper/)

## Update

*Note:* `kong-ingress-controller` v0.4 was released on April 24th; although I've not yet
had a chance to try it myself, it should render this helper obsolete as it sets
"preserve_host" to true by default now.

https://github.com/Kong/kubernetes-ingress-controller/blob/master/CHANGELOG.md#040---20190424

## Introduction

This is a simple helper designed to solve a particular integration problem
facing the co-operative working of the `kong-ingress-controller` and `cert-manager`.

At the time of writing (March '19; `kong-ingress-controller` v0.3, `cert-manager` v0.6),
the ACME http01 challenge doesn't quite work as it should. The reason for this
is that the kong ingress controller, by default, sets "preserve_host" to false
when creating new routes, but the challenge endpoint automatically created
by cert-mananger needs the host preserved in order to complete the verification.

More info here:
<https://github.com/jetstack/cert-manager/issues/958>

The issue should go away when cert-manager implements this feature request:
<https://github.com/jetstack/cert-manager/issues/1097>
but we don't know when that will be (was slated for v0.7 but looking like it won't make it)

I contemplated hacking the ingress controller to treat the acme ingresses as special,
but in the end decided to write a small helper tool as a temporary workaround.

It's designed to run as a sidecar to the ingress controller. It watches for new ingresses that
are created with the `cm-acme-http-solver-*` pattern. When one is created, it starts polling kong
until it sees the corresponding route created.
It then patches the route to set preserve_host to true. It continues to monitor the route and patch
if needed (as there's a possibility the ingress controller might undo the change - I'm not sure)
until the route dissapears, which is a sign that the challenge has been successfully completed.
It then goes back to quietly watching for new ingresses.

It's only had limited testing, but has thus far been successful in allowing http01 challenges
to complete automatically.

## Deploying it

Deploy it inside the `kong-ingress-controller` pod, as a third container (it already has the
ingress controller and the admin api). Something as simple as:

```yaml
      - name: acme-kong-kube-helper
        image: ollystephens/acme-kong-kube-helper:0.0.2
        imagePullPolicy: IfNotPresent
```

This should work if you've used the default yaml file for the ingress crontroller. If not,
you might need to tweak the arguments to let it know the admin api url.

The helper needs to run with equivalent permissions to the ingress controller;
it needs to be able to listen for ingress creation events across the whole cluter. The
easiest way to give it the correct permissions is to include it in the same pod, as suggested
above. If you don't want to do that, the second easiest thing to do is make sure it uses
the same `serviceaccount` as the ingress controller.

## Arguments

The first two arguments mirror their equivalents in `kong-ingress-controller`; if you are
deploying this container alongside, as suggested above, they should reflect the same settings
if present, or be left unset if not.
The third argument defines the name pattern for the ingress controllers created by `cert-manager`

| FLAG               | Meaning                                 | Default Value           |
| ------------------ | --------------------------------------- | ----------------------- |
| `-kong-url`        | URL for Kong admin api server           | `http://localhost:8001` |
| `-ingress-class`   | Ingress class being routed through Kong | `kong`                  |
| `-ingress-pattern` | Glob pattern for ingress name           | `cm-acme-http-solver-*` |

## Watching it in action

Watching the log of the container whilst you add a 'Certificate' object in should show
something like:

```log
2019/03/15 13:48:41 Matching ingress added: cm-acme-http-solver-fpkj9
2019/03/15 13:48:41   path /.well-known/acme-challenge/a1b2c3
2019/03/15 13:48:51 found matching kong route: /.well-known/acme-challenge/a1b2c3 = d4e5-f6e7
2019/03/15 13:48:51 successfully patched kong route: d4e5-f6e7
2019/03/15 13:49:01 found matching kong route: /.well-known/acme-challenge/a1b2c3 = d4e5-f6e7
2019/03/15 13:49:01 nothing to do; route for /.well-known/acme-challenge/a1b2c3 already has preserve_host set
2019/03/15 13:49:31 mission accomplished for path /.well-known/acme-challenge/a1b2c3
```
