# Windows Exporter Role

This role installs and configures the Prometheus Windows Exporter on Windows hosts.

## Variables

- `windows_exporter_version`: The version to install
- `windows_exporter_arch`: Usually "amd64"
- `windows_exporter_installer_url`: URL of the .msi file
- `windows_exporter_installer_path`: Local destination on the Windows machine

## Example Playbook

```yaml
- name: Install Windows Exporter
  hosts: windows_exporters
  roles:
    - windows_exporter
