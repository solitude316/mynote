# 

# 解決 KVM 無法使用 virsh console 連線到 Ubuntu VM

## 說明

雖然 KVM 已經在 XML 中配置了 console 通道，但 Ubuntu 預設不會自動將系統的登入題是字串 (getty) 綁到虛擬機的序列埠上，所以無法使用 `virsh console [domain]`   來登入虛擬機。

## 解決步驟

#### 步驟一：檢查 Host 的 XML 設定

使用 virsh 檢查 VM 的 XML 設定：

```bash
virsh edit [domain_name]
```

找到有關 pty 的設定是否正確：

```xml
<serial type='pty'>
  <target type='isa-serail' port='0'>
    <model name='isa-serial' />
  </target>
</serial>
<console type='pty'>
  <target type='serial' port='0'>
</console>
```

#### 步驟二：啟用 Serial Getty 服務

登入 VM 後，先啟用 serial-getty 服務：

```bash
sudo systemctl enable serial-getty@ttyS0.service
sudo systemctl start serial-getty@ttyS0.service
```

#### 步驟三：修改 GRUB 設定

確保核心 (Kernel) 在開機時，會將訊息輸出到序列埠。

編輯開啟設定檔`/etc/default/grub`，找到 GRUB_CMDLINE_LINUX_DEFAULT 及 GRUB_CMDLINE_LINUX 兩個參數：

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200"
```

參數說明如下：

- `GRUB_CMDLINE_LINUX_DEFAULT` 的設定會套用到**一般開機模式**。

- `GRUB_CMDLINE_LINUX` 的設定會套用到所有開機模式，包含**一般開機**與**救援模式 (Recovery Mode)**。

#### 步驟四：重啟 VM

設定完成，重啟 VM 後，就可以順利使用 `virsh console` 登入主機。


