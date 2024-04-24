# Howto setup DevSpaces to connect to Bitbucket using ssh keys for Authentication

## Prerequisites

* Create a personal private/public ssh key pair

> **IMPORTANT:** for use with DevSpaces, the ssh key ***MUST NOT*** have a passphrase, otherwise automatic checkout of the Git/Bitbucket repository by the DevSpaces workspaces will fail

* upload the public key to your user in Bitbucket

## Configuration Steps for DevSpaces

* Login to OpenShift using the user running the DevSpaces workspace.
* Assumption: DevSpaces uses the standard OpenShift namespace pattern '<username>-devspaces' for the user specific workspace

```shell

SSH_KEY=/path/to/ssh/private-key
SSH_PUB_KEY=/path/to/ssh/public-key

GITUSER=your-user-id
GITEMAIL=your-email@volkswagen.de

OCUSER=$(oc whoami)
SSHCONFIGTMP=/tmp/${OCUSER}_ssh_config
GITCONFIGTMP=/tmp/${OCUSER}_git_config
DEVSPACESNS=$(oc whoami)-devspaces



cat <<EOF > $SSHCONFIGTMP
host *
  IdentityFile /etc/ssh/dwo_ssh_key
  StrictHostKeyChecking = no
EOF

cat <<EOF > $GITCONFIGTMP
[user]
	name = ${GITUSER}
	email = ${GITEMAIL}
EOF


oc create secret -n "$DEVSPACESNS" generic git-ssh-key \
  --from-file=dwo_ssh_key="$SSH_KEY" \
  --from-file=dwo_ssh_key.pub="$SSH_PUB_KEY" \
  --from-file=ssh_config=$SSHCONFIGTMP

oc patch secret -n "$DEVSPACESNS" git-ssh-key --type merge -p \
  '{
    "metadata": {
      "labels": {
        "controller.devfile.io/mount-to-devworkspace": "true",
        "controller.devfile.io/watch-secret": "true"
      },
      "annotations": {
        "controller.devfile.io/mount-path": "/etc/ssh/",
        "controller.devfile.io/mount-as": "subpath"
      }
    }
  }'

oc create secret -n "$DEVSPACESNS" generic git-config \
  --from-file=gitconfig=$GITCONFIGTMP


oc patch secret -n "$DEVSPACESNS" git-config --type merge -p \
  '{
    "metadata": {
      "labels": {
        "controller.devfile.io/mount-to-devworkspace": "true",
        "controller.devfile.io/watch-secret": "true"
      },
      "annotations": {
        "controller.devfile.io/mount-path": "/etc/",
        "controller.devfile.io/mount-as": "subpath"
      }
    }
  }'

```

## Usage

* Create the Devspaces Workspace
* Clone the Bitbucket repository, e.g. `ssh://git@devstack.vwgroup.com:7999/dpm/rhintegration.git`

## Cleanup Secrets (in case you need to change content)

Cleanup (delete) the secrets in case new contents is required. Afterwards, perform the setup steps above again. Set the environment variables as required (see setup above).

```shell
oc delete secret -n "$DEVSPACESNS" git-ssh-key
oc delete secret -n "$DEVSPACESNS" git-config
```


## References

See as well: https://github.com/devfile/devworkspace-operator/blob/main/docs/additional-configuration.adoc#configuring-devworkspaces-to-use-ssh-keys-for-git-operations



