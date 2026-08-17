# GPU role

This role performs an explicitly requested NVIDIA driver package installation
from `apply-versions.yml`. It never selects a package automatically, changes the
kernel, installs the CUDA Toolkit, or reboots a node.

The shared `detect.yml` task file is read-only and is also used by the audit
role to distinguish NVIDIA PCI hardware, the loaded driver, and CUDA Toolkit.
