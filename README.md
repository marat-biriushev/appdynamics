# AppDynamics Machine Agent

Installs the AppDynamics Machine Agent (standalone bundle with its own
JRE) as a systemd service and points it at the controller
`appd-prd-app01.ipa.ipotekabank.uz:443`. Targets RHEL with systemd.

The agent does not touch the ESB services. Roll out in order
**dev -> test -> prod**, and let the AppDynamics team confirm the hosts
registered in the controller before moving to the next environment.

## Usage

1. Put the bundle on the control node (it is gitignored):

   ```
   files/machineagent-bundle-64bit-linux-26.8.0.5560.zip
   ```

   The servers do not need internet access - the zip is copied from the
   control node.

2. Provide the access key for the environment (see below).

3. Run, environment by environment - one host first, then the rest:

   ```bash
   ansible-playbook -i inventories/esb-dev.ini machine-agent.yml -k -K --ask-vault-pass \
     --limit accountdom-dev-srv01.ipa.ipotekabank.uz
   ansible-playbook -i inventories/esb-dev.ini machine-agent.yml -k -K --ask-vault-pass
   ```

   Hosts must be in the `appdynamics` group.

## Inventories

| Inventory | Group | Vault file |
|---|---|---|
| `inventories/esb-dev.ini` | `esb_dev` | `inventories/group_vars/esb_dev/vault.yml` |

Each environment group is a child of `appdynamics`. Its `vars.yml` maps
`machine_agent_access_key` to `vault_machine_agent_access_key`, which
lives in the vaulted `vault.yml` next to it.

## What it does

Mirrors the manual install from the ticket:

1. installs `unzip`
2. creates the `appdynamics` system group and user (`/sbin/nologin`,
   no home)
3. creates `/opt/appdynamics/machine-agent` owned by `appdynamics`
4. unpacks the bundle there and chowns it to `appdynamics:appdynamics`
5. writes `conf/controller-info.xml` (mode `0640`, holds the access
   key); `unique-host-id` is the server's `hostname` output, case kept
6. installs `/etc/systemd/system/appdynamics-machine-agent.service`
   from the unit shipped in the bundle, with these lines forced:

   ```
   Environment=MACHINE_AGENT_HOME=/opt/appdynamics/machine-agent
   Environment=JAVA_HOME=/opt/appdynamics/machine-agent/jre
   User=appdynamics
   ```

   `Group=` is set to `appdynamics` too if the unit has one. The rest of
   the unit is kept as shipped. The role fails if the shipped unit has
   no line to set.
7. `daemon-reload`, enables and starts `appdynamics-machine-agent`

Changes to the bundle, `controller-info.xml` or the unit restart the
agent. A second run changes nothing.

## Access key

`machine_agent_access_key` is required and differs per environment
(ESB-DEV / ESB-TEST / ESB-PROD). The role refuses to run without it.
Keep it in `ansible-vault`, per environment group:

```bash
ansible-vault create inventories/group_vars/esb_dev/vault.yml
# vault_machine_agent_access_key: "..."
```

Then add `--ask-vault-pass` (or `--vault-password-file`) to the run.

## Variables

| Variable | Default | Notes |
|---|---|---|
| `machine_agent_version` | `26.8.0.5560` | |
| `machine_agent_bundle` | `{{ playbook_dir }}/files/machineagent-bundle-64bit-linux-{{ machine_agent_version }}.zip` | path on the control node |
| `machine_agent_user` / `machine_agent_group` | `appdynamics` | |
| `machine_agent_home` | `/opt/appdynamics/machine-agent` | keep the default - the shipped unit's `ExecStart` points here |
| `machine_agent_java_home` | `{{ machine_agent_home }}/jre` | bundled JRE |
| `machine_agent_controller_host` | `appd-prd-app01.ipa.ipotekabank.uz` | |
| `machine_agent_controller_port` | `443` | |
| `machine_agent_controller_ssl` | `true` | |
| `machine_agent_account_name` | `customer1` | |
| `machine_agent_access_key` | `""` | **required**, vault |
| `machine_agent_sim_enabled` | `true` | Server Visibility |
| `machine_agent_unique_host_id` | `{{ ansible_facts['nodename'] }}` | same as `hostname`; not `ansible_hostname`, which drops the domain part |

## Upgrades

Drop the new zip into `files/`, bump `machine_agent_version`, run.
Each version leaves a marker `.bundle-<version>` in the install
directory; a missing marker unpacks the new bundle over the old one,
then `controller-info.xml` and the unit are re-applied and the agent
restarts.

## Dry run

`--check` on a host without the agent stops after the user/directory
steps: there is nothing unpacked to read the systemd unit from yet.
On hosts that already have the agent, `--check --diff` shows the full
diff.

## Verify on a host

```bash
systemctl status appdynamics-machine-agent
tail -f /opt/appdynamics/machine-agent/logs/machine-agent.log
```

Registration itself is confirmed on the controller side.
