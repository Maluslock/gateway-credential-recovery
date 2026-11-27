# 网关口令与配置取证方法论

> 从一台自有的中国电信天翼网关（天邑 / 海思平台）实机作业中，抽掉型号和凭据后留下的方法。
> 目标不是"破解密码学"，而是**找到固件里那段本来就会把明文口令吐出来的代码**。

![status](https://img.shields.io/badge/status-verified-brightgreen) ![scope](https://img.shields.io/badge/scope-own--device-blue) ![lang](https://img.shields.io/badge/docs-中文-red)

---

## 一句话结论

嵌入式网关的超级口令几乎无法靠猜，但**绝大多数固件都留着一条“喂文件名 → 吐明文”的本地 IPC 命令**。
你要做三件事，顺序不能反：

1. **面测绘**：分清 `200 / 302 / 403 / 404-400` 四档语义，找出“同一个固件的不同门”；
2. **提权**：找到 LAN 维护通道（telnet），拿到本地 shell；
3. **取证**：让设备自己解配置，而不是在本地逆 AES。

> 本体案例里，离线逆密钥花了整整一轮：1500 组弱 key + 13824 组 PBKDF2 + 67 组身份派生**全灭**，
> 最后靠一条 `hi_ipc ... decrypt_file` 两分钟解决。这就是本仓库存在的理由。

---

## 目录

| 文件 | 内容 |
|---|---|
| [docs/01-methodology.md](docs/01-methodology.md) | 五阶段流程（面测绘 → 鉴权模型 → 提权 → 数据获取 → 验证清场），含 10 条反面模式与可复用清单 |
| [docs/03-case-study-tewa7501v.md](docs/03-case-study-tewa7501v.md) | 完整案例：设备档案、三处卡点的正解、证据链、时间线（凭据全部脱敏） |
| [scripts/recover_superpass_template.py](scripts/recover_superpass_template.py) | 可复现脚本模板：开 telnet → su → 设备自解配置 → 分块搬回 → 提取口令 → 关 telnet |

---

## 五阶段

```
① 面测绘 ──→ ② 鉴权模型推断 ──→ ③ 提权（LAN shell）──→ ④ 数据获取 ──→ ⑤ 验证与清场
  端口/页面        会话 vs 角色           su/派生口令          让设备自解        双向验证
  端点命名规则      cookie/token 语义      假阴性排查           而非逆密钥        锁额度/关端口
```

**判据比动作重要**：本案例跑的三轮冤枉路，全部源于判据错——

| 现象 | 错误判读 | 真相 |
|---|---|---|
| `GET /xxx.cgi` → `400` | "接口不存在" | 该服务器只接受 **POST**，空 body 打一次就通了 |
| `echo code \| su -c id` → `Error!` | "su 口令错" | stdin 不是 tty 时 busybox **不打印 `Password:`**，静默 `exit 1` |
| 80 端口超管登录失败 | "口令错" | 80 只走 `CheckUAPasswd` 分支，超管入口在 **8080** |

---

## 核心技巧速览

**端点名不要猜，要从二进制里“读”**：

```bash
strings /usr/bin/web | grep '>>ENTER'      # → 290 个真实 CGI 处理器名
strings /bin/hipcbox  | head                # 有些“程序”其实是 /bin/sh 脚本
```

**配置加密的五个判定**（按顺序，快且不留遗憾）：魔数 → 压缩 → 单字节 XOR → 16B 块重复率 → 弱 key 字典。
若五连全灭且二进制里没有硬编码 key，就别再爆破了——去找设备自带的解密命令：

```bash
cat /home/cmd/encrypt/decrypt_file
#   functionname=hi_decrypt_file; attributenum=2;
#   alias<ifile>  alias<ofile>
hi_ipc /home/cmd/encrypt/decrypt_file -v ifile <加密配置> ofile /tmp/out.xml
```

**验证必须双向**：设备自带 oracle（DBus `Check*Passwd`）+ 真实 web 登录回执，两条路结果一致才写进交付物。
顺手先做一次**控制实验**（拿已知正确的普通账号登一次），否则你永远分不清“口令错”和“我请求写错了”。

---

## 适用范围与边界

- 只针对**自有设备**。本仓库不提供任何他人设备的攻击面。
- 全程**零写操作**：不改密码、不恢复出厂、不动 TR-069/LOID。
- 注意失败锁：web 常见 3 次/60s；telnet 授权锁常为 60s 且**全局**（多会话共享），连打只会把锁续期。
- 会重新随机化超密的按钮（如后台“维护结束”）不要点；真要长期稳定，方向是置空 ACS 地址而不是删 TR-069 连接。

---

## 许可

MIT。方法可自由参考，出问题别找我。
