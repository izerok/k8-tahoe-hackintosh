# 脱敏与许可

> 原 README §9。

## 9. 脱敏与许可

- SMBIOS（序列号/MLB/UUID）与 ROM/MAC 在 config 中为 `GENERATE-WITH-GenSMBIOS` / 全零；请用 GenSMBIOS 生成自己的
- 内网地址替换为 `192.0.2.x`（RFC 5737 文档网段），用户名替换为 `user`
- 第三方组件版权归各自作者，见 `THIRD_PARTY_NOTICES.md`；本仓库自有文字与配置以 MIT 发布
- 仅供学习研究；黑苹果与实验性 GPU 驱动有损坏系统/数据风险，操作前做好备份
