---
layout: "docs"
page_title: "Vault Agent Templates"
sidebar_title: "TemplatesAuto-Auth"
sidebar_current: "docs-agent-templates"
description: |-
  Vault Agent's Template functionality allows Vault secrets to be rendered to files 
  using Consul Template markup.
---

# Vault Agent Templates

Vault Agent's Template functionality allows Vault secrets to be rendered to files 
using [Consul Template markup](https://github.com/hashicorp/consul-template#templating-language).

## Functionality

The `template` stanza configures the templating engine in the Vault agent for rendering 
secrets to files using Consul Template markup language.  Multiple `template` stanzas 
can be defined to render multiple files.

When the agent is started with templating enabled, it will attempt to acquire a 
Vault token using the configured Method. On failure, it will back off for a short 
while (including some randomness to help prevent thundering herd scenarios) and 
retry. On success, secrets defined in the templates will be retrieved from Vault and 
rendered locally.

## Configuration

The top level `template` block has multiple configurations entries:

- `source` `(object: optional)` - Path on disk to use as the input template.  This 
option is required if not using the `contents` option.
- `destination` `(object: required)` - Path on disk where the rendered secrets should 
be created. If the parent directories If the parent directories do not exist, Vault 
Agent will attempt to create them, unless `create_dest_dirs` is false.
- `create_dest_dirs` `(object: required)` - This option tells Vault Agent to create 
the parent directories of the destination path if they do not exist. The default 
value is true.
- `contents` `(object: optional)` - This option allows embedding the contents of 
a template in the configuration file rather then supplying the `source` path to 
the template file. This is useful for short templates. This option is mutually 
exclusive with the `source` option.
- `command` `(object: optional)` - This is the optional command to run when the 
template is rendered. The command will only run if the resulting template changes. 
The command must return within 30s (configurable), and it must have a successful 
exit code. Vault Agent is not a replacement for a process monitor or init system.
- `command_timeout` `(object: optional)` - This is the maximum amount of time to 
wait for the optional command to return. Default is 30s.
- `error_on_missing_key` `(object: optional)` - Exit with an error when accessing 
a struct or map field/key that does notexist. The default behavior will print "<no value>" 
when accessing a field that does not exist. It is highly recommended you set this 
to "true". 
- `perms` `(object: optional)` - This is the permission to render the file. If 
this option is left unspecified, Vault Agent will attempt to match the permissions 
of the file that already exists at the destination path. If no file exists at that 
path, the permissions are 0644. 
- `backup` `(object: optional)` - This option backs up the previously rendered template 
at the destination path before writing a new one. It keeps exactly one backup. 
This option is useful for preventing accidental changes to the data without having 
a rollback strategy.
- `left_delimiter` `(object: optional)` - Delimiter to use in the template. The 
default is "{{" but for some templates, it may be easier to use a different 
delimiter that does not conflict with the output file itself. 
- `right_delimiter` `(object: optional)` - Delimiter to use in the template. The 
default is "}}" but for some templates, it may be easier to use a different 
delimiter that does not conflict with the output file itself.
- `sandbox_path` `(object: optional)` - If a sandbox path is provided, any path 
provided to the `file` function is checked that it falls within the sandbox path. 
Relative paths that try to traverse outside the sandbox path will exit with an error. 
- `wait` `(object: required)` - This is the `minimum(:maximum)` to wait before rendering 
a new template to disk and triggering a command, separated by a colon (`:`).

## Example Template

Template with Vault Agent requires the use of the `secret` [function from Consul Template](https://github.com/hashicorp/consul-template#secret).  
The following is an example of a template that retrieves a generic secret from Vault's 
KV store:

```
{{ with secret "secret/my-secret" }}
{{ .Data.data.foo }}
{{ end }}
```

## Example Configuration

The following demonstrates configuring Vault Agent to template secrets using the 
AppRole Auth method:

```python
pid_file = "./pidfile"

vault {
  address = "https://127.0.0.1:8200"
}

auto_auth {
  method {
    type      = "approle"

    config = {
      role_id_file_path = "/etc/vault/roleid"
      secret_id_file_path = "/etc/vault/secretid"
    }
  }

  sink {
    type = "file"

    config = {
      path = "/tmp/file-foo"
    }
  }
}

template {
  source      = "/tmp/agent/template.ctmpl"
  destination = "/tmp/agent/render.txt"
}
```
