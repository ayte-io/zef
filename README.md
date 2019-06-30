# Project draft

This is simply a text describing a project. The project itself hasn't 
been created yet.

# Zef

Zero-servered Chef.

## What is it?

Zef is an opinionated framework on top of 
[Knife Zero](https://knife-zero.github.io). It's target is to provide 
serverless approach for vanilla Chef in a very straightforward way with
minimum friction, cutting out some options.

## Motivation

First of all, handling separate Chef server is usually very cumbersome 
for small companies. One of major Ansible keys is it's 
infrastructureless, which allows single person to manage their hosts
without hassle and too many time spent on setting things up. So why
don't we do the same trick with Chef?

Second, Knife Zero is a very neat tool, but it comes with complexity,
repeatable actions and a lot of configuration options to be set. At one 
point it became too sad, so I've decided create a convention-first tool 
that would fulfill my (and hopefully not only my) needs.

## Usage

Zef follows repository-based approach. Repository contains nodes 
definitions, policyfiles and some state that has to be persisted between
Zef calls.

Everything starts with a Policyfile or a directory Policyfile resides
in: 

```bash
zef converge vps/jump-host
zef converge vps/jump-host/Policyfile.rb
zef converge vps/jump-host/Policyfile.lock.json
``` 

After receiving a command Zef starts looking for configuration file,
walking up directory tree until it finds one:

```yaml
# zef.yml
apiVersion: v1
kind: Configuration # Yeah, Kubernetes fan here
spec:
  behavior:
    update: false # Should policy files be updated on every call?
  state:
    attributes: # List of persisted attributes
      - node.name
  paths:
    workspace: workspace # Relative path to directory where policies will be exported
    content: src # Directory with policies and nodes
    state: state # Directory for persisted state 
```

Then it looks in target directory for a file in `zef` subdirectory,
called `zef/nodes.yml` 

```yaml
# zef/nodes.yml
apiVersion: v1
kind: NodeCollection
spec:
  attributes:
    default:
      ayte:
        zef:
          version: 0.1.1
  nodes:
    ds-01.srv.ayte.io: {}
    1.1.1.1:
      hostname: ds-02.srv.ayte.io
      attributes:
        override:
          ayte:
            zef:
              version: 0.1.0
```

Specification declares nodes eligible for policy application, as well as
their attributes.

Finally, the converge process takes place. Zef installs and updates
policy if necessary, exports it to workspace, then takes filtered subset 
of nodes and applies exported policy to each of them in a sequential 
manner (if you want to go parallel, just open several shell sessions).

NB: It is implied that one node can't be shared between two policies.

## CLI options

### Converge

Converges target node collection with optional filtering by pattern(s).

```
zef converge <path> [<pattern...>]

zef converge vps/jump-hosts/helsinki vps-01
```

| Name                   | Default      | Notes                |
|:----------------------:|:------------:|:--------------------:|
| --configuration        | `zef.yml`    |
| --knife-configuration  | none         |
| --update               | none         | Force policy update |
| --workspace            | config value |
| --connection-user      | `root`       |
| --connection-password  | none         |
| --connection-port      | `22`         |
| --connection-identity  | none         | SSH key to use for target machine
| --connection-protocol  | `ssh`        | `ssh` or `winrm`
| --connection-known-hosts-file | $HOME/.ssh/known_hosts | 
| --sudo                 | none         | no value
| --sudo-with-password   | none         | no value
| --ssh-gateway          | none         |
| --ssh-gateway-identity | none         | SSH key to use for jump host
| --overwrite            | none         | no value, no sense in stateless mode
| --log-level            | info         | `info` or `debug` 


### Forget

Erases node(s) state and bootstraps it (them) on next launch.

```
zef forget <path> [<pattern...>]

zef forget vps/jump-hosts/helsinki vps-01
```

| Name                   | Default      | Notes |
|:----------------------:|:------------:|:-----:|
| --configuration        | `zef.yml`    |       |
