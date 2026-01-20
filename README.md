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

8. Unmount image
```
cd # and close any terminals in the mounted image folder
sudo umount ~/mnt/img_p2
sudo losetup -d /dev/loop0
rmdir ~/mnt/img_p2
```
Your `crankkos-linxdotrk3566-1.0.0.img` file is now ready for flashing. 

### Flash image:
1. Install https://github.com/rockchip-linux/rkdeveloptool as per the instructions. You may need to add more steps based on your os. Check the github issues for help
```
git clone https://github.com/rockchip-linux/rkdeveloptool
cd rkdeveloptool
autoreconf -i
./configure
make
cp rkdeveloptool ~/.local/bin`
```
2. Plug in a usb-c cable to the device, hold down the power button while plugging in the power cable. Continue holding for 5 seconds. Check that the device shows up via `lsusb` etc.
3. Check that the device shows up in rkdeveloptool in `Loader` mode:
```
$ sudo rkdeveloptool ld
DevNo=1	Vid=0x2207,Pid=0x350a,LocationID=307	Loader
```
4. Erase the flash, then reconnect power plug:
```
sudo rkdeveloptool ef
Erasing flash complete.
# Now reconnect power plug
```

5. Check that the device is in Maskrom mode
```
sudo rkdeveloptool ld
DevNo=1	Vid=0x2207,Pid=0x350a,LocationID=307	Maskrom
```
6. Flash the bootloader from https://github.com/fernandodev/linxdot-rockchip-flash/
```
wget https://github.com/fernandodev/linxdot-rockchip-flash/raw/refs/heads/main/rk356x_spl_loader_ddr1056_v1.10.111.bin
```
```
sudo rkdeveloptool db rk356x_spl_loader_ddr1056_v1.10.111.bin
Downloading bootloader succeeded.
```
7. Flash .img file
```
sudo rkdeveloptool cs 1
Change Storage OK.
```
```
sudo rkdeveloptool wl 0 crankkos-linxdotrk3566-1.0.0.img 
Write LBA from file (100%)
```
```
sudo rkdeveloptool td
Test Device OK.
```
```
sudo rkdeveloptool rd
Reset Device OK.
```

### Configure Image (root password and wifi):
1. SSH into the device and set a password. Ignore the readonly errors. The root password is `crankk`
```
ssh root@x.x.x.x
passwd
```
2. Add wifi credentials:
`/data/etc/wpa_supplicant.conf`
```
update_config=1
ctrl_interface=/var/run/wpa_supplicant
network={
    scan_ssid=1
    ssid="xxxxx"
    psk="xxxx"
}
```
3. Additional startup scripts can be added to `/data/etc/userinit.sh`
