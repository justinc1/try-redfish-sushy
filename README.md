- https://docs.openstack.org/sushy-tools/latest/user/dynamic-emulator.html
- https://github.com/openstack/sushy-tools/blob/master/doc/source/install/index.rst
- https://uefi.org/sites/default/files/resources/UEFI_Plugfest_May_2015_HTTP_Boot_Redfish_Samer_El-Haj_ver1.2.pdf

```
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

tmpfile=$(mktemp /tmp/sushy-domain.XXXXXX)
virt-install \
   --name vbmc-node \
   --ram 1024 \
   --disk size=1 \
   --vcpus 2 \
   --os-type linux \
   --os-variant fedora28 \
   --graphics vnc \
   --print-xml > $tmpfile
virsh define --file $tmpfile

sushy-emulator --libvirt-uri qemu:///session

curl http://localhost:8000/redfish/v1/Systems/

curl -d '{"ResetType":"On"}' \
    -H "Content-Type: application/json" -X POST \
     http://localhost:8000/redfish/v1/Systems/vbmc-node/Actions/ComputerSystem.Reset
curl -d '{"ResetType":"ForceOff"}' \
    -H "Content-Type: application/json" -X POST \
     http://localhost:8000/redfish/v1/Systems/vbmc-node/Actions/ComputerSystem.Reset

```

UEFI boot via ipxe

```
tmpfile=$(mktemp /tmp/sushy-domain.XXXXXX)
virt-install \
   --name vbmc-node \
   --ram 1024 \
   --boot loader.readonly=yes \
   --boot loader.type=pflash \
   --boot loader.secure=no \
   --boot loader=/usr/share/OVMF/OVMF_CODE.secboot.fd \
   --boot nvram.template=/usr/share/OVMF/OVMF_VARS.fd \
   --disk size=1 \
   --vcpus 2 \
   --os-type linux \
   --os-variant fedora28 \
   --graphics vnc \
   --print-xml > $tmpfile
virsh define --file $tmpfile

cat <<EOF >emulator.conf
SUSHY_EMULATOR_BOOT_LOADER_MAP = {
    'Uefi': {
        'x86_64': '/usr/share/OVMF/OVMF_CODE.secboot.fd',
    }
}
SUSHY_EMULATOR_SECURE_BOOT_ENABLED_NVRAM = '/usr/share/OVMF/OVMF_VARS.secboot.fd'
SUSHY_EMULATOR_SECURE_BOOT_DISABLED_NVRAM = '/usr/share/OVMF/OVMF_VARS.fd'
EOF

sushy-emulator --libvirt-uri qemu:///session --config $PWD/emulator.conf


curl -d '{"Boot": {"BootSourceOverrideTarget":"Pxe"}}' \
     -H "Content-Type: application/json" \
 -X  PATCH http://localhost:8000/redfish/v1/Systems/47d7f5bd-c871-41a6-9e84-94d3c3fb09e5


```
