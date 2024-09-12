# Ansible Role: Informix Management

An [Ansible Galaxy](https://galaxy.ansible.com/) role for provisioning management scripts, cron jobs, and an interactive menu system for working with Informix databases at the command-line.

## Table of contents

* [Role Variables][1]
    * [Informix Cron Jobs][2]
* [Interactive Menu][3]
* [Common Management Scripts][4]
* [Example Requirements File][5]
* [Example Playbook][6]
* [License][7]

[1]: #role-variables
[2]: #informix-cron-jobs
[3]: #interactive-menu
[4]: #common-management-scripts
[5]: #example-requirements-file
[6]: #example-playbook
[7]: #license

## Role Variables

The following role variables are required, however default values may be used where applicable:

| Name                                    | Default    | Description                                            |
|-----------------------------------------|------------|--------------------------------------------------------|
| `informix_management_alerts_enabled`    | `false`    | A boolean value representing whether to enable the retrieval of alerts configuration from Hashicorp Vault. If enabled (i.e. set to `true`), configuration will be retrieved from Hashicorp Vault using the path specified by the `informix_management_alerts_vault_path` variable. This configuration will be stored in a variable named `informix_management_alerts_config` and can be referenced in Jinja2 templates. |
| `informix_management_alerts_vault_path` |            | The Hashicorp Vault path to read alerts configuration from when `informix_management_alerts_enabled` is true. This configuration is made accessible to Jinja2 template scripts installed from the `informix_management_script_templates_path` path via the template variable `informix_management_alerts_config`. |
| `informix_management_cron_jobs`         |            | See [Informix Cron Jobs][2] for more information.      |
| `informix_management_informix_group`    | `informix` | The group to be used for ownership of script files, directories, and cron jobs. |
| `informix_management_informix_user`     | `informix` | The user to be used for ownership of script files and directories. |
| `informix_management_install_path`      | `/opt/informix/14.10` | The path to the Informix installation directory. |
| `informix_management_logs_path`         | `/var/log/informix` | The path to a directory which will be created for storing the output of cron job scripts. |
| `informix_management_script_templates_path` |            | _Optional_. An optional path to a directory containing zero or more [Jinja2](https://jinja.palletsprojects.com/en/2.10.x/) format template script files. These scripts will be installed to `/home/{{ informix_management_informix_user }}/scripts/` and can be executed as cron jobs. See [Informix Cron Jobs][2] for more information. In addition, several common Informix DB management scripts are installed in the same location (see [Common Management Scripts][4]). |
| `informix_management_stats_enabled`     | `false`    | A boolean value representing whether to enable the retrieval of stats configuration from Hashicorp Vault. If enabled (i.e. set to `true`), configuration will be retrieved from Hashicorp Vault using the path specified by the `informix_management_stats_vault_path` variable. This configuration will be stored in a variable named `informix_management_stats_config` and can be referenced in Jinja2 templates. |
| `informix_management_stats_vault_path`  |            | The Hashicorp Vault path to read stats configuration from when `informix_management_stats_enabled` is true. This information is made accessible to Jinja2 template scripts installed from the `informix_management_script_templates_path` path via the template variable `informix_management_stats_config`. |

### Informix Cron Jobs

The `informix_management_cron_jobs` variable can be used to schedule cron jobs for the user defined by the `informix_management_informix_user` variable. This is primarily used to perform level zero and continuous logical log backups for Informix databases. If unset, _no_ cron jobs will be configured. This variable should be defined as a list, and each item in this list represents a single cron job for the Informix user defined by the variable `informix_management_informix_user`. The following parameters are required for each item in the list:

| Name                 | Default | Description                                                                          |
|----------------------|---------|--------------------------------------------------------------------------------------|
| `day_of_month`       |         | Day of the month the job should run (`1-31`, `*`, `*/2`, and so on).                 |
| `day_of_week`        |         | Day of the week that the job should run (`0-6` for Sunday-Saturday, `*`, and so on). |
| `disabled`           | `false` | If the job should be disabled in the crontab (`true` or `false`).                    |
| `hour`               |         | Hour when the job should run (`0-23`, `*`, `*/2`, and so on).                        |
| `minute`             |         | Minute when the job should run (`0-59`, `*`, `*/2`, and so on).                      |
| `month`              |         | The month that the job should run.                                                   |
| `name`               |         | A description of the job. This parameter should be unique across all jobs defined for a given group. |
| `script`             |         | The name of the script to execute. A Jinja2 template for this script is expected to exist within the directory specified by the `informix_management_script_templates_path` variable, with a `.j2` extension. The `script` parameter value should not include the `.j2` file extension. |

For example, to execute the `level_zero_backup` script at midnight every day, passing the script the single argument `ef`:

```yaml
informix_management_cron_jobs:
  - name: Perform level zero backup for EF database
    day_of_week: "*"
    day_of_month: "*"
    month: "*"
    minute: "0"
    hour: "0"
    script: "level_zero_backup ef"
```

## Interactive Menu

An interactive shell function named `menu` will be installed by this role, and the `.bash_profile` configuration file for the user specified by the `informix_management_informix_user` variable will be updated to automatically execute this function when a new shell is created. The menu presented to the user will include an option for each database server configuration on the host, and when an option is selected the current shell will be configured for that specific database server. This is intended for use with tools like `dbaccess`, which require configuration to be present in the environment. The menu can be invoked again at any time by running the `menu` function on the command-line, and will reconfigure the current shell for the database server selected by the user.

# Common Management Scripts

This role deploys a number of common management scripts for regular Informix DB maintenance tasks in addition to those scripts provided by the `informix_management_script_templates_path` role variable:

| Filename                      | Description                                                                                                         |
|-------------------------------|---------------------------------------------------------------------------------------------------------------------|
| `check_continuous_backups.j2` | Check for the presence of continuous logical log backup processes and generate email alerts if neccessary.          |
| `level_zero_backup.j2`        | Perform a level zero database backup and archive the level zero backup file.                                        |
| `logging.j2`                  | Common logging functions for use in other scripts.                                                                  |
| `logical_log_archive.j2`      | Perform logical log backup and restart continuous backups for the specified Informix database.                      |
| `stop_all_logicals.j2`        | Stop all logical log continuous backup processes.                                                                   |
| `update_statistics.j2`        | Generate and update database statistics regarding table, row, and page-count in the systables system catalog table. |

> [!WARNING]
> To avoid clashes, scripts provided using the `informix_management_script_templates_path` role variable should not be given the same name as any of the common management scripts listed above.

All scripts are installed to the directory path `/home/{{ informix_management_informix_user }}/scripts/`.

## Example Requirements File

```yml
---

collections:
  - name: companieshouse.middleware
    version: "1.0.0"
```

## Example Playbook

```yml
- name: Provision Informix DB management tooling
  hosts: all
  roles:
    - role: companieshouse.middleware.informix_management
```

## License

This project is subject to the terms of the [MIT License](LICENSE).
