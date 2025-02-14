- Modify boot up EL
  https://stackoverflow.com/questions/42824706/qemu-system-aarch64-entering-el1-when-emulating-a53-power-up
- Emulating aarch64 Linux
  https://blog.jitendrapatro.me/emulating-aarch64arm64-with-qemu-part-1/
- Change GIC version
  use `-machine virt,gic-version=$(GIC_VERSION)`
  - GIC_VERSION can be 2, 3, 4