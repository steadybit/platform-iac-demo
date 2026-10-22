# Steadybit as code: onboard a team

Onboard a team to [Steadybit](https://www.steadybit.com) from files in Git, with the
[Steadybit CLI](https://github.com/steadybit/cli): its environment, the team with its members
and allowed actions, its service, and a first experiment. Each top-level folder is one team,
as a project `steadybit apply -d` understands.

```
checkout/
  environments/checkout.yml            what the team may see and attack (a target query)
  teams/checkout.yml                   members, roles, allowed environments and actions
  services/checkout.yml                the service, on the service profile "Steadybit Starter"
  experiments/checkout-downstream.yml  an experiment from a template, for the service
  values/checkout-downstream.yml       the template's placeholder values
.github/workflows/chaos.yml            runs the team's experiments in GitHub Actions
```

## Prerequisites

- A Kubernetes cluster, e.g. [minikube](https://minikube.sigs.k8s.io).
- The Steadybit agent in that cluster, installed with the cluster name `minikube`: the
  files select targets by `k8s.cluster-name`, which comes from the agent's configuration.
- The example application [Shopping Demo](https://github.com/steadybit/shopping-demo),
  deployed as described in
  [Deploy Example Application](https://docs.steadybit.com/quick-start/deploy-example-application),
  which puts it in the namespace `steadybit-shopping-demo` the files expect.
- The [Steadybit CLI](https://github.com/steadybit/cli#installation) with an admin access
  token: environments and teams need one.

With another cluster name or namespace, change them in the environment, the service, and the
template values.

## Onboard the team

The team's members in `checkout/teams/checkout.yml` are examples: replace them with the email
addresses of users who can already log in to your tenant. Steadybit skips members it doesn't
know, so an address with a typo leaves that person out of the team without an error.

```bash
steadybit apply -d checkout --dry-run   # what would be created
steadybit apply -d checkout             # the environment, the team, the service, the experiment
```

`apply` writes the ids and keys of the new objects back into the files.

## A baseline from the service profile

The service uses the service profile "Steadybit Starter", which recommends experiments for
it. Create the ones that fit, run them, and watch the service's risk:

```bash
SERVICE=$(sed -n 's/^id: //p' checkout/services/checkout.yml)
steadybit service experiment list -i "$SERVICE"              # recommended, created or not
steadybit service experiment provide -i "$SERVICE" --template <template-id>
steadybit experiment run -k <experiment-key>
steadybit service risk -i "$SERVICE"
```

Provided experiments stay out of Git: exported, they are plain experiment files with the
service's id in them, so they wouldn't carry over to another team.

## An experiment from a template

`checkout/experiments/checkout-downstream.yml` is the template "Unavailable downstream service
doesn't result in user-visible errors" with `checkout/values/checkout-downstream.yml` filled in.
To create it from the template instead, and link it to the service:

```bash
steadybit template get -i 0363a90e-9563-494c-bd25-81da6e4c3d43 --placeholders
steadybit experiment apply --template 0363a90e-9563-494c-bd25-81da6e4c3d43 \
  --team CHK --environment Checkout --external-id chk-downstream \
  --placeholders checkout/values/checkout-downstream.yml
steadybit service experiment link -i "$SERVICE" -k <experiment-key> --category Dependency
```

Run it every Monday at 7:00:

```bash
steadybit schedule create -k <experiment-key> --cron "0 0 7 ? * MON" --timezone Europe/Berlin
```

## Drift

When someone changes the team's setup in the Steadybit UI, `diff` shows it and exits with 2,
so a scheduled job can catch it. `apply` puts back what is in Git.

```bash
steadybit diff -d checkout
steadybit apply -d checkout
```

## The team's pipeline

Create a token for the team and store it as the repository secret `STEADYBIT_TOKEN`, without
printing it:

```bash
steadybit access-token create --name ci-checkout --type TEAM --team CHK -t json --jq .token \
  | gh secret set STEADYBIT_TOKEN
```

Add `pull_request:` to the triggers in `.github/workflows/chaos.yml` to run the experiments on
every pull request; branch protection in GitHub then blocks a merge until they pass.

## The next team

Copy `checkout/` to a new folder, remove the `id:` and `key:` lines `apply` wrote, change the
team's key, name and members, the environment and service queries, and the template values,
and apply the new folder.

## Known issues

- A team file of more than about 8 KB is refused with an HTML `403 Forbidden`. A team allowed
  every action is that big; list only the actions the team needs.
- `diff` reports a difference for an environment file with only `query`, because the
  platform also returns the query as `predicate.query`. Add a `predicate` with the same
  `query` to avoid that.
- `diff` reports a difference for `allowedActions` in another order than the platform's. The
  team file lists them the way `steadybit team get` writes them.
- `diff` reports a difference for team members listed by email only, because the platform
  adds their user id.
