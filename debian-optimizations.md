# Debian Optimizations

```
echo 'vm.swappiness=10' | tee /etc/sysctl.d/99-swappiness.conf
sysctl --system

apt install zram-tools
```
