# Data Hub Jupyterhub Runbook

This document provides information on how to administer our Jupyterhub
deployment.

## Key Locations

Production Namespace: [dh-prod-jupyterhub](https://datahub.psi.redhat.com/console/project/dh-prod-jupyterhub/overview)
Stage Namespace: [dh-stage-jupyterhub](https://paas.stage.psi.redhat.com/console/project/dh-stage-jupyterhub/overview)

## Issue Escalation with PSI

If we need to escalate an issue with the PSI team, instructions for doing so can be found at [PnT Devops - Issue Escalation Protocol](https://mojo.redhat.com/docs/DOC-1049381).

## Prerequisites

Our Jupyterhub instance is managed by the [Open Data Hub operator][1]. The ODH
Operator needs to be deployed in the pertinent Openshift Namespace by the
cluster-admin.

## Deployment

The Jupyterhub server is deployed using Kustomize. As such, any changes should be made
in the necessary files in git and redeployed (rather than manually editing
OpenShift objects directly).

Please read the [README.md](https://github.com/AICoE/aicoe-sre/blob/master/applications/jupyterhub/README.md) for deployment instructions.

## Upkeep and Administration

### Admin Tool

Jupyterhub includes an Admin tool that lets us manage users. In production,
this tool can be reached [here][2]. Note that your user must
be granted admin rights to be able to access this page. The list of users
with this access is maintained in [vars/prod-vars.yaml](vars/prod-vars.yaml)
in the `jupyterhub_admins` list. The admin tool will let you stop/start,
delete, and access a user's notebook server and can be very helpful for
addressing issues that Jupyterhub users may encounter.

### Custom Notebook Sizes

We frequently get requests from users to be able to spin up notebook pods
with more compute resources than what our default sizes allow. The
configuration to enable this is maintained in the `jupyter-singleuser-profiles`
configmap for the Jupyterhub deployment. We specify additional profiles for the
amount of memory or cpu that the user is requesting as well as the user that
has access to those resources. This configuration is managed in git via TBD.

Once a configuration change has been made to customize a user's resources
that user will need to restart their notebook pod for the changes to take
effect. **Important:** The user will need to select the default size when
starting their notebook (rather than one of our preset sizes)

### Custom Spark Cluster Size

Users can deploy a Spark cluster alongside their notebook pod. Occasionally a
user will need a spark cluster with more compute resources than our default
configuration. Instructions for how to do this will be added at a later date.

## Common Problems

The following is a list of common issues we've encountered with Jupyterhub and
how to fix them.

### Insufficient disk space for notebook pod

When a user runs out of disk space on their notebook pod, the pod will fail
to start and give little indication to the user about why that's happening.
User our [grafana dashboard][3] to determine if the user has, in fact, used up
their storage quota.

If the user has run out of storage space, we have an ansible playbook that
can be run to recover from this situation. It does so by backing up the data
in the user's PVC, creating a new (larger) PVC, and restoring the data from
the backup. This playbook can be found [here][5].

To run the recovery playbook, perform the following (from the root of your
local clone of the dh-jupyterhub repo):

```bash
pipenv shell
pipenv install
cd user_pvc_recovery
ansible-playbook recover_user_pvc_data.yaml \
  -e kubeconfig=$HOME/.kube/config \
  -e target_env=prod
```

*Note* Upon running the playbook you will be prompted to enter the kerberos
ID of the user whose PVC you want to recover as well as a desired size for
the new (larger) PVC. By default, PVCs are sized at 2GB. A common option for
larger PVC sizes is 5GB. The user may request a significantly larger PVC. If
this is the case, ask the user to justify why they need so much space and why
they are unable to use Ceph S3 object storage instead of local notebook
storage.

### User unable to start server

Jupyterhub occasionally gets into a corrupted state where it thinks that the notebook pod for a user is running when it actually isn't running. When the user tells Jupyterhub to take the server down, they will either get stuck in an infinite loop or get an error saying that the pod isn't running. The fix is as follows:

1. Login as a Data Hub admin user to the jupyterhub namespace
2. Check to see if the user's pod shows up in the list of running pods. If it does, delete it.
3. Delete the _jupyterhub_ pod. Do __NOT__ delete the _jupyterhub-db_ pod.
4. Ask the user to login again. They should be able to spin up their notebook server.

### Insufficient hardware to satisfy notebook pod requests

This issue happens most often when a user requests a GPU for their notebook
pod and all GPUs are in use. This results in the pod not starting. To solve
this either have the user launch their notebook without a GPU (if they don't
need one), or figure out who is using the GPUs and if they're actively using
them. If they are not then we can stop their notebook pods to free up the GPUs
they are using. This is best done using the admin tool (see above).

A useful command to figure out who is using the gpus courtesy of Pete Mackinnon

```bash
oc get pods -o json --all-namespaces | jq -r '.items[] | select(.spec.containers[0].resources.requests["nvidia.com/gpu"]>=1 and .status.phase=="Running") | .metadata.name + "\n" + .metadata.namespace + "\n" + .spec.nodeName + " \n Request GPU: " + .spec.containers[0].resources.requests["nvidia.com/gpu"] + " \n Limit GPU: " + .spec.containers[0].resources.limits["nvidia.com/gpu"] + "\n"'
```

## Smoke Test

To Consider Jupyterhub as `UP` and `Available` go to the
[Jupyterhub endpoint][4]. Verify that the page loads, that you can log in, and
that you can spin up a notebook.

## Alerts

TODO

[1]: https://gitlab.cee.redhat.com/data-hub/dh-internal-odh-install
[2]: https://jupyterhub.datahub.redhat.com/hub/admin
[3]: https://grafana.datahub.redhat.com/dashboard/db/jupyterhub-user-storage-capacity?refresh=30s&orgId=1
[4]: https://jupyterhub.datahub.redhat.com
[5]: https://gitlab.cee.redhat.com/data-hub/dh-jupyterhub/blob/master/user_pvc_recovery/recover_user_pvc_data.yaml
