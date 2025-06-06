# SELinux and Zabbix

SELinux (Security-Enhanced Linux) provides mandatory access control for Zabbix by
enforcing security policies that restrict what the Zabbix processes can do, even
when running as root.

SELinux contexts are a core component of how SELinux implements security control.
Think of contexts as labels that are assigned to every object in the system (files, processes, ports, etc.).
These labels determine what can interact with what.

### SELinux Enforcement Mode

For SELinux to actually provide security protection, it needs to be set to "enforcing" mode. There are three possible modes for SELinux:

- **Enforcing** - SELinux security policy is enforced. Actions that violate policy are blocked and logged.
- **Permissive** - SELinux security policy is not enforced but violations are logged. This is useful for debugging.
- **Disabled** - SELinux is completely turned off.

You can check the current SELinux mode with the getenforce command:

```yaml
getenforce
```

This should return : Enforcing

To properly secure Zabbix with SELinux, the system should be in `Enforcing` mode. If it's not, you can change it temporarily:

##### Set to enforcing immediately (until reboot)

```yaml
sudo setenforce 1
```

For permanent configuration, edit /etc/selinux/config and set:

```yaml
SELINUX=Enforcing
```

### Basic Structure of an SELinux Context

An SELinux context typically consists of four parts:

- **User**: The SELinux user identity (not the same as Linux users)
- **Role**: What roles the user can enter
- **Type**: The domain for processes or type for files (most important part)
- **Level**: Optional MLS (Multi-Level Security) sensitivity level

When displayed, these appear in the format: user:role:type:level

### How Contexts Work in Practice

In the Zabbix SELinux configuration, several security types are defined to control access:

- **zabbix_t**: The domain in which the Zabbix server process runs
- **zabbix_port_t**: Type assigned to network ports that Zabbix uses
- **zabbix_var_run_t**: Type for Zabbix runtime socket files
- **httpd_t**: The domain for the Apache web server process

The SELinux policy allows specific permissions between these types:

Zabbix server can connect to its own Unix stream sockets
Zabbix server can connect to network ports labeled as zabbix_port_t
Zabbix server can create and remove socket files in directories labeled as zabbix_var_run_t

The web server (httpd) can connect to Zabbix ports, allowing the web frontend to
communicate with the Zabbix server.
These permissions ensure Zabbix components can communicate properly while maintaining
SELinux security boundaries.

When Zabbix tries to access a file or network resource, SELinux checks if the context
of the Zabbix process is allowed to access the context of that resource according to
policy rules.

### Viewing Contexts

!!! info "You can view the contexts of files using:"

    ```yaml
    ls -Z /path/to/zabbix/files
    ```

!!! info "And for the processes:"

    ```yaml
    ps -eZ | grep zabbix"
    ```
    ``` yaml
    system_u:system_r:unconfined_service_t:s0 691 ?  00:02:20 zabbix_agent2
    system_u:system_r:zabbix_t:s0       707 ?        00:00:59 zabbix_server
    system_u:system_r:zabbix_t:s0      1203 ?        00:02:00 zabbix_server
    ```

!!! info "And for log files"

    ``` yaml
    ls -alZ /var/log/zabbix/zabbix_server.log
    ```yaml
    -rw-rw-r--. 1 zabbix zabbix system_u:object_r:zabbix_log_t:s0 11857 Apr 26 22:02 /var/log/zabbix/zabbix_server.log
    ```

### Zabbix-selinux-policy Package

The zabbix-selinux-policy package is a specialized SELinux policy module designed
specifically for Zabbix deployments. It provides pre-configured SELinux policies
that allow Zabbix components to function properly while running in an SELinux enforced
environment.

Key Functions of the Package:

- **Pre-defined Contexts** : Contains proper SELinux context definitions for Zabbix
  binaries, configuration files, log directories, and other resources.
- **Port Definitions** : Registers standard Zabbix ports (like 10050 for agent, 10051 for server)
  in the SELinux policy so they can be used without triggering denials.
- **Access Rules**: Defines which operations Zabbix processes can perform, like writing
  to log files, connecting to databases, and communicating over networks.
- **Boolean Toggles**: Provides SELinux boolean settings specific to Zabbix that can
  enable/disable certain functionalities without having to write custom policies.

Benefits of Using the Package:

- **Simplified Deployment** : Reduces the need for manual SELinux policy adjustments when
  installing Zabbix.
- **Security by Default**: Ensures Zabbix operates with minimal required permissions rather than running in permissive mode.
- **Maintained Compatibility**: The package is updated alongside Zabbix to ensure compatibility with new features.

#### Installation and Usage

The package is typically installed alongside other Zabbix components:

```yaml
dnf install zabbix-selinux-policy
```

After installation, the SELinux contexts are automatically applied to standard Zabbix
paths and ports. If you use non-standard configurations, you may still need to make
manual adjustments.
This package essentially bridges the gap between Zabbix's operational requirements
and SELinux's strict security controls, making it much easier to run Zabbix securely
without compromising on monitoring capabilities.

### For Zabbix to function properly with SELinux enabled:

Zabbix binaries and configuration files need appropriate SELinux labels (typically zabbix_t context)
Network ports used by Zabbix must be properly defined in SELinux policy
Database connections require defined policies for Zabbix to communicate with MySQL/PostgreSQL
File paths for monitoring, logging, and temporary files need correct contexts

When issues occur, they typically manifest as denied operations in SELinux audit logs. Administrators can either:

Use audit2allow to create custom policy modules for legitimate Zabbix operations
Apply proper context labels using semanage and restorecon commands
Configure boolean settings to enable specific Zabbix functionality

This combination creates defense-in-depth by ensuring that even if Zabbix is compromised,
the attacker remains constrained by SELinux policies, limiting potential damage to
your systems.

#### Zabbix SELinux Boolean

One of the most convenient aspects of the SELinux implementation for Zabbix is the
use of "booleans". simple on/off switches that control specific permissions. These
allow you to fine-tune SELinux policies without needing to understand complex policy
writing. Key Zabbix booleans include:

- **zabbix_can_network**: Controls whether Zabbix can initiate network connections
- **httpd_can_connect_zabbix**: Controls whether the web server can connect to Zabbix
- **zabbix_run_sudo**: Controls whether Zabbix can execute sudo commands

!!! info "You can view these settings with:"
`yaml
    getsebool -a | grep zabbix
   `
And you can toggle them as needed with setsebool.

### Enable Zabbix network connections (persistent across reboots)

```yaml
setsebool -P zabbix_can_network on
```

These booleans make it much easier to securely deploy Zabbix while maintaining SELinux
protection, as you can enable only the specific capabilities that your Zabbix implementation
needs without compromising overall system security.

### Creating custom rules

When running Zabbix in environments with SELinux enabled, you may encounter permission
issues when Zabbix attempts to execute certain utilities like fping. This occurs
because fping uses setuid (SUID) permissions, and SELinux's default policies prevent
Zabbix from executing such binaries for security reasons.

There are different solutions to this problem:

- **Method 1: Automated Policy Generation :**

The most straightforward approach is to use the audit2allow utility to analyse
SELinux denial messages and generate appropriate policies:

First, capture the denial events from the audit log:

```yaml
sudo grep zabbix /var/log/audit/audit.log | grep fping | audit2allow -M zabbix_fping
```

!!! info "Install the generated policy module:"

    ```yaml
    sudo semodule -i zabbix_fping.pp
    ```

!!! info "Apply the correct SELinux context to the fping binary:"

    ```yaml
    sudo chcon -t fping_exec_t /usr/sbin/fping
    ```

- **Method 2: Manual Policy Creation :**

For more control or in situations where audit logs aren't available, you can manually
create a custom policy:

Create a policy file named zabbix_fping.te with the following content:

```yaml
module zabbix_fping 1.0;

require {
    type zabbix_t;
    type fping_t;
    type fping_exec_t;
    class file { execute execute_no_trans getattr open read };
    class capability net_raw;
}

#============= zabbix_t ==============
allow zabbix_t fping_exec_t:file { execute execute_no_trans getattr open read };
allow zabbix_t self:capability net_raw;
```

!!! info "Compile the policy module:"

    ```yaml
    checkmodule -M -m -o zabbix_fping.mod zabbix_fping.te
    ```

!!! info "Package the compiled module:"

    ```yaml
    semodule_package -o zabbix_fping.pp -m zabbix_fping.mod
    ```

!!! info "Install the policy module:"

    ```yaml
    semodule -i zabbix_fping.pp
    ```

## Securing zabbix admin

## HTTPS

## DB certs

## Conclusion

## Questions

- Why does SELinux prevent Zabbix from executing fping by default?
- In what situations might you need to create custom SELinux policies for other Zabbix monitoring tools?
- What are the key differences between using audit2allow and manually creating a custom policy module?

## Useful URLs

- https://www.zabbix.com/documentation/7.2/en/manual/installation/install_from_packages/rhel?hl=SELinux#selinux-configuration
- https://www.systutorials.com/docs/linux/man/8-zabbix_selinux/
- https://man.linuxreviews.org/man8/zabbix_agent_selinux.8.html
- https://phoenixnap.com/kb/selinux
