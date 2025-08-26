# OS Ubuntu Rsyslog Log Forwarding Configuration

## Introduction
This document provides a detailed, technical guide for configuring Rsyslog on an Ubuntu system to forward logs to a remote server in RFC5424 format over UDP port 5140.


## Configuring Rsyslog for Log Forwarding
To configure Rsyslog to forward logs in RFC5424 format to a remote server over UDP port 5140, follow these steps. The configuration will be implemented in a modular file within `/etc/rsyslog.d/` to ensure maintainability and compatibility with Ubuntu’s configuration management practices.

### Step 1: Verify Rsyslog Installation
Rsyslog is typically pre-installed on Ubuntu systems. Verify its presence and operational status using:

```bash
sudo systemctl status rsyslog
```

This command checks if the Rsyslog service is active and running. The output should indicate `active (running)`.

If Rsyslog is not installed, install it with:

```bash
sudo apt-get update
sudo apt-get install rsyslog
```

**Verification**:
- Confirm the installation by checking the Rsyslog version:

```bash
rsyslogd -v
```

- Ensure the service is enabled to start on boot:

```bash
sudo systemctl enable rsyslog
```

### Step 2: Create a Configuration File in /etc/rsyslog.d/
To maintain a modular configuration, create a new configuration file in the `/etc/rsyslog.d/` directory, which is included by the main `/etc/rsyslog.conf` file via the `$IncludeConfig /etc/rsyslog.d/*.conf` directive. Files in this directory are processed in lexicographical order, so naming the file with a low numerical prefix like `10-` ensures it is loaded early, prioritizing log forwarding for cybersecurity purposes.

Create the file using a text editor:

```bash
sudo nano /etc/rsyslog.d/10-remote-forwarding.conf
```

**Verification**:
- Ensure the `/etc/rsyslog.d/` directory exists:

```bash
ls /etc/rsyslog.d/
```

- Confirm that `/etc/rsyslog.conf` includes the directory by checking for the line:

```bash
$IncludeConfig /etc/rsyslog.d/*.conf
```

This line is typically present in default Ubuntu configurations.

### Step 3: Configure Log Forwarding in /etc/rsyslog.d/10-remote-forwarding.conf
In the `10-remote-forwarding.conf` file, configure queue settings for reliability and specify the forwarding rule using the RFC5424 format. The RFC5424 format ensures compatibility with modern syslog servers, providing structured logging with high-precision timestamps and standardized fields.

Add the following content to `/etc/rsyslog.d/10-remote-forwarding.conf`:

```bash
# Define working directory for queue storage
$WorkDirectory /var/spool/rsyslog

# Define RFC5424 template for structured logging
$template RFC5424Format,"<%PRI%>1 %TIMESTAMP:::date-rfc3339% %HOSTNAME% %APP-NAME% %PROCID% %MSGID% %STRUCTURED-DATA% %msg%\n"

# Configure queue settings for reliable log delivery
$ActionQueueFileName fwdRule1
$ActionQueueMaxDiskSpace 1g
$ActionQueueSaveOnShutdown on
$ActionQueueType LinkedList
$ActionResumeRetryCount 1000

# Forward all messages to the remote server on UDP port 5140
*.* @remote-server-ip:5140;RFC5424Format
```

**Detailed Explanation**:
- **Working Directory**:
  - `$WorkDirectory /var/spool/rsyslog`: Specifies the directory where Rsyslog stores temporary queue files. This ensures that buffered logs are saved to disk if the remote server is unavailable, preventing data loss.
- **Queue Settings**:
  - `$ActionQueueFileName fwdRule1`: Assigns a unique identifier for the queue.
  - `$ActionQueueMaxDiskSpace 1g`: Limits the queue to 1GB of disk space to prevent excessive storage use.
  - `$ActionQueueSaveOnShutdown on`: Saves queued logs to disk on service shutdown, ensuring no data loss.
  - `$ActionQueueType LinkedList`: Uses an in-memory linked list for efficient queue management.
  - `$ActionResumeRetryCount -1`: Configures infinite retries for sending logs, ensuring delivery attempts continue until successful.
- **Forwarding Rule**:
  - `*.* @remote-server-ip:5140;RSYSLOG_SyslogProtocol23Format`: Forwards all log messages (all facilities and severities) to the specified remote server IP address over UDP port 5140 using the `RSYSLOG_SyslogProtocol23Format` template, which is compatible with RFC5424. The single `@` denotes UDP; use `@@` for TCP if desired.

**Customization**:
- Replace `remote-server-ip` with the actual IP address of the remote log server (e.g., `192.168.1.100`).
- To forward specific logs (e.g., authentication logs only), modify the rule to `auth.* @remote-server-ip:5140;RSYSLOG_SyslogProtocol23Format`.

**Verification**:
- Validate the configuration syntax before saving:

```bash
sudo rsyslogd -N1 -f /etc/rsyslog.d/10-remote-forwarding.conf
```

- Ensure the file has appropriate permissions:

```bash
sudo chmod 644 /etc/rsyslog.d/10-remote-forwarding.conf
sudo chown root:root /etc/rsyslog.d/10-remote-forwarding.conf
```

### Step 4: Save and Restart Rsyslog
Save the changes to `/etc/rsyslog.d/10-remote-forwarding.conf` and restart the Rsyslog service to apply the configuration:

```bash
sudo systemctl restart rsyslog
```

Verify that the service is running without errors:

```bash
sudo systemctl status rsyslog
```

**Verification**:
- Check the system logs for Rsyslog errors:

```bash
sudo tail -f /var/log/syslog
```

- Confirm that the configuration files are loaded correctly by checking Rsyslog’s debug output (if needed):

```bash
sudo rsyslogd -d
```

### Step 5: Verify Log Forwarding
To confirm that logs are being forwarded correctly:
- Use a network packet capture tool on the remote server to verify UDP traffic on port 5140:

```bash
sudo tcpdump -i any -n port 5140
```

- Generate a test log message on the client system to verify forwarding:

```bash
logger "Test message for remote logging"
```

**Verification**:
- Ensure the remote server is configured to accept UDP syslog messages on port 5140 and supports RFC5424 format.
- Check for firewall rules that might block UDP port 5140 on either the client or server:

```bash
sudo ufw status
```


## Best Practices and Security Considerations
- **Network Security**: Restrict access to UDP port 5140 on the remote server using firewall rules or a VPN. For example, use `ufw` to allow only specific IP addresses:

```bash
sudo ufw allow from client-ip to any port 5140 proto udp
```


- **Testing**: Validate the configuration in a non-production environment to confirm compatibility with the remote server and log format.

## Troubleshooting
- **Logs Not Forwarding**:
  - Verify configuration syntax: `sudo rsyslogd -N1`.
  - Ensure the remote server is reachable: `ping remote-server-ip`.
  - Confirm the server is listening on UDP port 5140: `netstat -ulnp | grep 5140`.
- **Firewall Issues**: Check for blocking rules on the client or server:

```bash
sudo ufw status
```

- **Format Issues**: Verify that the remote server supports RFC5424. If it requires a different format, adjust the template in `10-remote-forwarding.conf`.
- **Queue Issues**: If logs are not delivered, check the queue directory (`/var/spool/rsyslog`) for buffered logs and ensure sufficient disk space.

## Conclusion
This guide enables system administrators to configure Rsyslog on Ubuntu to forward logs in RFC5424 format to a remote server over UDP port 5140, using a modular configuration in `/etc/rsyslog.d/10-remote-forwarding.conf`. By leveraging key log types like `/var/log/auth.log` and `/var/log/syslog`, administrators can enhance security monitoring to detect and respond to threats effectively. The configuration is robust, secure, and aligned with industry standards, making it suitable for enterprise-grade centralized logging solutions.

**Citations**:
- [Rsyslog Documentation: Templates](https://www.rsyslog.com/doc/configuration/templates.html)
- [RFC5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [Red Hat Enterprise Linux 9 Security Hardening Guide](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/security_hardening/assembly_configuring-a-remote-logging-solution_security-hardening)
- [How to Set Up Centralized Logging on Linux with Rsyslog](https://betterstack.com/community/guides/logging/how-to-configure-centralised-rsyslog-server/)