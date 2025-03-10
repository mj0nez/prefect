---
description: Learn the basics of troubleshooting Prefect.
tags:
    - troubleshooting
    - guides
    - how to
---

# Troubleshooting

Don't Panic! If you experience an error with Prefect, there are many paths to understanding and resolving it. The first troubleshooting step is confirming that you are running the latest version of Prefect. If you are not, be sure to [upgrade](#upgrade) to the latest version, since the issue may have already been fixed. Beyond that, there are several categories of errors:

* The issue may be in your flow code, in which case you should carefully read the [logs](#logs).
* The issue could be with how you are authenticated, and whether or not you are connected to [Cloud](#cloud).
* The issue might have to do with how your code is [executed](#execution).


## Upgrade

Prefect is constantly evolving, adding new features and fixing bugs. Chances are that a patch has already been identified and released. Search existing [issues](https://github.com/PrefectHQ/prefect/issues) for similar reports and check out the [Release Notes](https://github.com/PrefectHQ/prefect/blob/main/RELEASE-NOTES.md). Upgrade to the newest version with the following command:

```bash
pip install --upgrade prefect
```

Different components may use different versions of Prefect:

* **Cloud** will generally always be the newest version. Cloud is continuously deployed by the Prefect team. When using a self-hosted server, you can control this version.
* **Agents and workers** typically don't change versions frequently, and are usually whatever the latest version was at the time of creation. Agents and workers provision infrastructure for flow runs, so upgrading them may help with infrastructure problems.
* **Flows** could use a different version than the agent or worker that created them, especially when running in different environments. Suppose your agent and flow both use the latest official Docker image, but your agent was created a month ago. Your agent will often be on an older version than your flow.

!!! note "Integration Versions"
    Keep in mind that [integrations](/integrations/) are versioned and released independently of the core Prefect library. They should be upgraded simultaneously with the core library, using the same method.


## Logs

In many cases, there will be an informative stack trace in Prefect's [logs](/concepts/logs/). **Read it carefully**, locate the source of the error, and try to identify the cause.

There are two types of logs:

* **Flow and task logs** are always scoped to a flow. They are sent to Prefect and are viewable in the UI.
* **Agent and worker logs** are not scoped to a flow and may have more information on what happened before the flow started. These logs are generally only available where the agent or worker is running.

If your flow and task logs are empty, there may have been an infrastructure issue that prevented your flow from starting. Check your agent logs for more details.

If there is no clear indication of what went wrong, try updating the logging level from the default `INFO` level to the `DEBUG` level. [Settings](/api-ref/prefect/settings/) such as the logging level are propagated from the agent or worker environment to the flow run environment and can be set via environment variables or the `prefect config set` CLI:

```bash
# Using the CLI
prefect config set PREFECT_LOGGING_LEVEL=DEBUG

# Using environment variables
export PREFECT_LOGGING_LEVEL=DEBUG
```

The `DEBUG` logging level produces a high volume of logs so consider setting it back to `INFO` once any issues are resolved.


## Cloud

When using Prefect Cloud, there are the additional concerns of authentication and authorization. The Prefect API authenticates users and service accounts - collectively known as actors - with API keys. Missing, incorrect, or expired API keys will result in a 401 response with detail `Invalid authentication credentials`. Use the following command to check your authentication, replacing `$PREFECT_API_KEY` with your API key:

```bash
curl -s -H "Authorization: Bearer $PREFECT_API_KEY" "https://api.prefect.cloud/api/me/"
```

!!! note "Users vs Service Accounts"
    [Service accounts](/cloud/users/service-accounts/) - sometimes referred to as bots - represent non-human actors that interact with Prefect such as agents, workers, and CI/CD systems. Each human that interacts with Prefect should be represented as a user. User API keys start with `pnu_` and service account API keys start with `pnb_`.

Supposing the response succeeds, let's check our authorization. Actors can be members of [workspaces](/cloud/workspaces/). An actor attempting an action in a workspace they are not a member of will result in a 404 response. Use the following command to check your actor's workspace memberships:

```bash
curl -s -H "Authorization: Bearer $PREFECT_API_KEY" "https://api.prefect.cloud/api/me/workspaces"
```

!!! note "Formatting JSON"
    Python comes with a helpful [tool](https://docs.python.org/3/library/json.html#module-json.tool) for formatting JSON. Append the following to the end of the command above to make the output more readable: ` | python -m json.tool`

Make sure your actor is a member of the workspace you are working in. Within a workspace, an actor has a [role](/cloud/users/roles/) which grants them certain permissions. Insufficient permissions will result in an error. For example, starting an agent or worker with the __Viewer__ role, will result in errors.


## Execution

Prefect flows can be executed locally by the user, or remotely by an agent or worker. Local execution generally means that you - the user - run your flow directly with a command like `python flow.py`. Remote execution generally means that an agent or worker runs your flow via a [deployment](/concepts/deployments/), optionally on different infrastructure.

With remote execution, the creation of your flow run happens separately from its execution. Flow runs are assigned to a work pool and a work queue. In order for flow runs to execute, an agent or worker must be subscribed to the work pool and work queue, otherwise the flow runs will go from `Scheduled` to `Late`. Ensure that your work pool and work queue have a subscribed agent or worker.

Local and remote execution can also differ in their treatment of relative imports. If switching from local to remote execution results in local import errors, try replicating the behavior by executing the flow locally with the `-m` flag (i.e. `python -m flow` instead of `python flow.py`). Read more about `-m` [here](https://stackoverflow.com/a/62923810).
