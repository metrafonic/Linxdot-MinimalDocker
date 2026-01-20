# Linxdot-MinimalDocker
A minimal Buildroot system image for Linxdot RK3566 Helium miners (LD1001)

### Credits
- [crankkio](https://github.com/crankkio) - Reverse engineering the Linxdot and creating the buildroot image.

## Manual Configuration

### Prepare .img
1. Download the [crankkos-linxdotrk3566-1.0.0.img.xz](https://crkk1.spaces.crankk.net/crankkos/crankkos-linxdotrk3566-1.0.0.img.xz) firmware
```
wget https://crkk1.spaces.crankk.net/crankkos/crankkos-linxdotrk3566-1.0.0.img.xz
xz -d crankkos-linxdotrk3566-1.0.0.img.xz
```

2. Mount image for modifications
```
sudo losetup --find --show --partscan crankkos-linxdotrk3566-1.0.0.img
lsblk /dev/loop0
mkdir -p ~/mnt/img_p2
sudo mount -o rw /dev/loop0p2 ~/mnt/img_p2
~/mnt/img_p2
```

3. Replace the existing `./etc/docker-compose.yml` with the following. Customize the region if needed (can be changed later via the `/data/opt/packet_forwarder/configs/` config)
```yml
version: '3'
services:
  pktfwd:
    image: ghcr.io/heliumdiy/sx1302_hal:sha-87d8931
    container_name: pktfwd
    entrypoint: ["/opt/packet_forwarder/entrypoint.sh"]
    working_dir: /opt/packet_forwarder
    environment:
      VENDOR: linxdot
      REGION: EU868
    privileged: true
    volumes:
      - /opt/packet_forwarder/tools:/opt/packet_forwarder/tools:ro
      - /data/opt/packet_forwarder/configs:/opt/packet_forwarder/configs
    restart: always

  miner:
    image: quay.io/team-helium/miner:gateway-latest
    container_name: miner
    command: ["helium_gateway", "server"]
    restart: always
    environment:
      GW_KEYPAIR: "ecc://i2c-5:96?slot=0"
      GW_ONBOARDING: "ecc://i2c-5:96?slot=15"
      GW_REGION: "EU868"
      GW_LISTEN: "0.0.0.0:1680"
      RUST_BACKTRACE: "1"
    devices:
      - "/dev/i2c-5:/dev/i2c-5:rwm"
    cap_add:
      - SYS_RAWIO
```

4. add the LDO reset script:
```
mkdir -p ./opt/packet_forwarder/tools
wget -o ./opt/packet_forwarder/tools/reset_lgw.sh.linxdot https://github.com/metrafonic/Linxdot-MinimalDocker/raw/refs/heads/main/reset_lgw.sh.linxdot
chmod +x ./opt/packet_forwarder/tools/reset_lgw.sh.linxdot
```

6. Set the default root password to to `crankk`. I have generated a shadow file with this
```
echo "root:$1$rC7EtsDo$G9Thoj1.V4GjpNLYmuIRM.:::::::" > ./usr/share/dataskel/etc/shadow
```

7. Clean up the crontab
```
echo "13 * * * * /usr/sbin/logrotate /etc/logrotate.conf" > ./etc/crontabs/root
```
