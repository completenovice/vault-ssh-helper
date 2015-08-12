Vault SSH Agent
===============

Vault SSH Agent is a counterpart to Vault's (https://github.com/hashicorp/vault)
SSH backend.

Vault authenticated users create SSH OTPs to connect to remote hosts. During SSH
connection establishment, the keyboard-interactive password prompt receives the
OTP entered by the user and provides it to agent. Agent authenticates clients by
verifying the OTP with Vault server.

For enabling Vault OTP authentication, agent needs to be installed on all the hosts.
SSHD configuration should be modified to enable keyboard-interactive authentication.
SSHD PAM configuration should be modified to redirect client authentication to agent.

Usage
-----
### Options
| Option | Description |
|--------|-------------|
| `verify`| To verify that the agent is installed correctly and is able to talk to Vault successfully.
| `config-file` | The path to the configuration file. The properties of config file are mentioned below.

**[Note]: Refer the below configuration for Linux. It will differ for each platform.**
Agent Configuration
--------------------------------

Name the config file with `.hcl` extension (**`vault.hcl`**)

1) Address of the Vault server

**`VAULT_ADDR="http://127.0.0.1:8200"`**

2) SSH backend mount point in Vault server

**`SSH_MOUNT_POINT="ssh"`**

3) Path to PEM encoded CA Cert file used to verify Vault server SSL certificate

**`CA_CERT=""`**

4) Path to directory of PEM encoded CA Cert files used to verify Vault server
SSL Certificate.

**`CA_PATH=""`**

5) Skip TLS certificate verification. Highly not recommended.
(Boolean)

**`TLS_SKIP_VERIFY=false`**

PAM Configuration
--------------------------------

**`/etc/pam.d/sshd`**

1) Comment out the previous authentication mechanism. "common-auth" represents
the standard linux authentication module and is used by many applications.
Do not modify "common-auth" file.

**`#@include common-auth`**

2) pam_exec.so runs external commands. The external command in this case is
vault-ssh-agent. If the agent binary terminates with exit code 0 if authentication
is successful. If not it fails with exit code 1.

**`auth requisite pam_exec.so expose_authtok log=/tmp/vaultssh.log /usr/local/bin/vault-ssh-agent -config-file=/etc/vault/vault.hcl`**

'requisite' indicates that, if the external command fails, the authentication
 should fail.

'expose_authtok' provides the binary access to the password entered by the user.

3) Proper return from authentication flow. There will be a pipe open which listens
to the response of the authentication. Unfortunately, pam_exec.so is not closing
this pipe and pam_unix is.

**`auth optional pam_unix.so no_set_pass use_first_pass nodelay`**

'optional' indicates that this is not a mandatory step for authentication to succeed.

'no_set_pass' indicates that this module should not be allowed to set or modify passwords.

'use_first_pass' avoids the extra prompt and takes the OTP entered for keyboard-interactive
as its input and tries to authenticate.

'nodelay' avoids the induced terminal delay in case of authentication failure.

SSHD Configuration
--------------------------------

**`/etc/ssh/sshd_config`**

1) Enable challenge response authentication. This is nothing but keyboard-interactive
authentication.

**`ChallengeResponseAuthentication yes`**

2) Enabling this option is mandatory. Without this PAM authentication modules will
not be triggered.

**`UsePAM yes`**

3) [Optional] Disable password authentication. Both keyboard-interactive and
password mechanisms prompts for "Password", which can be confusing. In other
words, first few "Password" prompts belong to keyboard-interactive mechanism.
If all the responses are invalid, then the mechanism switches to password
authentication. Again, there will be "Password" prompts. Only this time, the
responses are checked against the password database at the target machine.

**`PasswordAuthentication no`**


Developing Vault-ssh-agent
---------------------------

If you wish to work on agent itself or any of its built-in systems, you'll
first need [Go](https://www.golang.org) installed on your machine
(version 1.4+ is required).

For local dev first make sure Go is properly installed, including setting up a
[GOPATH](https://golang.org/doc/code.html#GOPATH). After setting up Go, you can
download the required build tools such as vet, gox, godep etc by bootstrapping
your environment.

```sh
$ make bootstrap
...
```

Next, clone this repository into `$GOPATH/src/github.com/hashicorp/vault-ssh-agent`.
Then type `make`. This will run the tests. If this exits with exit status 0,
then everything is working 

```sh
$ make
...
```

To compile a development version of Vault-ssh-agent, run `make dev`. This will put
the vault-ssh-agent binary in `bin` and `$GOPATH/bin` folders:

```sh
$ make dev
...
$ bin/vault-ssh-agent
...
```

If you're developing a specific package, you can run tests for just that
package by specifying the `TEST` variable. For example below, only
`agent` package tests will be run.

```sh
$ make test TEST=./agent
...
```

