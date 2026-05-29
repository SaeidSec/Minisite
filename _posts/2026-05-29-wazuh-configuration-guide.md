---
title: Getting Started with Wazuh SIEM Configuration
date: 2026-05-29 11:00:00 +0000
categories: [SIEM, Wazuh]
tags: [wazuh, siem, configuration, security]
---

# Getting Started with Wazuh SIEM Configuration

Wazuh is a free and open-source platform for threat detection, incident response, and compliance. In this guide, we'll walk through the essential configuration steps to get your Wazuh deployment operational.

## Prerequisites

- Wazuh Manager and Agent installed
- Basic Linux/Windows system administration knowledge
- Network connectivity between manager and agents

## 1. Wazuh Manager Configuration

### Accessing the Web Interface

1. Open your browser and navigate to `https://your-wazuh-manager-ip`
2. Default credentials: `admin:admin` (change immediately in production)
3. Navigate to the management section

### Adding Agents

1. Go to **Management > Agents**
2. Click **Add new agent**
3. Follow the wizard for your OS (Windows, Linux, macOS)
4. Execute the installation commands on your target systems

## 2. Configuring Log Collection

Edit `/var/ossec/etc/ossec.conf` on the manager:

```xml
<ossec_config>
  <localfile>
    <location>/var/log/auth.log</location>
    <log_format>syslog</log_format>
  </localfile>
  
  <localfile>
    <location>/var/log/apache2/error.log</location>
    <log_format>apache</log_format>
  </localfile>
</ossec_config>
```

## 3. Creating Detection Rules

Rules are the backbone of Wazuh detection. Navigate to `/var/ossec/etc/rules/` and create custom rules:

```xml
<group name="custom_app,">
  <rule id="100001" level="3">
    <if_sid>syscheck</if_sid>
    <file_integrity_changed>/etc/important_config</file_integrity_changed>
    <description>Important configuration file changed</description>
  </rule>
</group>
```

## 4. Setting Up Alerts

Configure alerting thresholds:
- **Level 1-3**: Low severity events
- **Level 4-7**: Medium severity events
- **Level 8-15**: High severity events requiring investigation

## 5. Integration with External Tools

Wazuh can integrate with:
- Slack
- PagerDuty
- Splunk
- ELK Stack
- Custom webhooks

## 6. Dashboard Customization

Create custom dashboards for:
- Top alert categories
- Failed login attempts
- File integrity changes
- System resource usage
- Network anomalies

## Best Practices

1. **Start with baseline tuning** - Understand your environment before tuning alerts
2. **Reduce false positives** - Tune rules to your environment
3. **Regular backups** - Back up configuration and database
4. **Monitor the monitor** - Set alerts for Wazuh health
5. **Use aggregation** - Group related events for better visibility

## Common Issues

**Agents not connecting**: Check network connectivity and firewall rules
**High CPU usage**: Review rule count and optimize queries
**Storage issues**: Implement appropriate data retention policies

---

**Next Steps**: Check back for advanced Wazuh deployment strategies and threat hunting techniques!

Have questions about Wazuh configuration? Reach out on Twitter [@SaeidSec](https://twitter.com/SaeidSec) or connect on LinkedIn!
