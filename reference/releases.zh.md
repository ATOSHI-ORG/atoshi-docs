# 发布与版本

Atoshi 如何给版本打 tag、构建 `atoshid` 二进制,并发布它,以便节点运维者和升级流程都能获取一个可校验的构建。

## 版本策略

**tag 必须是 `v20.x.y`。** Go 模块路径是
`github.com/atoshi-chain/atoshi/v20`,因此 Go 的语义导入版本规则要求每个发布
tag 的主版本号必须是 **`v20`**。像 `v1.0.0` 这样的 tag 对 `/v20` 模块是无效的
—— `go get` 和依赖解析会忽略它。

- 下一个版本是 **`v20.3.0`** —— 它同时匹配模块路径*和*代码内的升级名 `v20.3`
  (`app/upgrades/v20_3/`),于是二进制版本、git tag、cosmovisor 升级目录、以及
  链上 `MsgSoftwareUpgrade` 的升级名四者对齐。
- 现有的 **`v1.0.0`** tag 对 `/v20` 模块无效,应当删除(或视为废弃)。不要用它
  构建发布。

```bash
# 退役这个无效 tag(本地 + 远端)
git tag -d v1.0.0
git push origin :refs/tags/v1.0.0
```

## 从源码构建

```bash
git clone https://github.com/atoshi-chain/atoshi
cd atoshi
git checkout v20.3.0
make build                      # → ./build/atoshid
./build/atoshid version --long  # 确认 Version = 20.3.0
```

`make build` 通过 ldflags(`.../version.Version`)把版本从 `git describe --tags`
里烙进二进制,因此从 `v20.3.0` tag 构建出的二进制会报告 `20.3.0`。从未打 tag 的
提交构建则报告短 commit hash。

## 发布一个版本

先打 tag,再发一个带**预编译二进制 + 校验和**的 GitHub Release,这样运维者不必
人人自己编译。

```bash
git tag -a v20.3.0 -m "v20.3.0"
git push origin v20.3.0
```

仓库已自带 `.goreleaser.yml`,它为五个目标构建 `atoshid` ——
`darwin/amd64`、`darwin/arm64`、`linux/amd64`、`linux/arm64`、`windows/amd64`
—— 并生成 `checksums.txt`:

```bash
# 需要 goreleaser + 一个具备 repo 权限的 GITHUB_TOKEN
goreleaser release --clean
```

它会为该 tag 创建 GitHub Release 并附上归档包和 `checksums.txt`。如果你想手动
做,就构建 `linux/amd64`(运维者需要的目标)并连同校验和一起上传:

```bash
make build
tar -czf atoshid_20.3.0_Linux_amd64.tar.gz -C build atoshid
shasum -a 256 atoshid_20.3.0_Linux_amd64.tar.gz > checksums.txt
gh release create v20.3.0 atoshid_20.3.0_Linux_amd64.tar.gz checksums.txt \
  --title "v20.3.0" --notes "…"
```

至少要发布 **`linux/amd64`** —— 那是验证人和全节点运行的目标(见
[节点运维](./node-operation.md))。

## 升级用的二进制 URL

`MsgSoftwareUpgrade` 提案和 cosmovisor 都需要一个 `upgrade-info`,指向新版本的
可下载二进制。Release 发布后,使用 release 资产的 URL:

```json
{"binaries":{
  "linux/amd64":"https://github.com/atoshi-chain/atoshi/releases/download/v20.3.0/atoshid_20.3.0_Linux_amd64.tar.gz?checksum=sha256:<sum>"
}}
```

cosmovisor 会在升级高度自动下载并切换到它。完整提案流程见
[治理操作手册](./governance-runbook.md#6-worked-example--software-upgrade-msgsoftwareupgrade)。

## 校验下载的二进制

```bash
# 校验和必须与发布的 checksums.txt 一致
shasum -a 256 atoshid_20.3.0_Linux_amd64.tar.gz
tar -xzf atoshid_20.3.0_Linux_amd64.tar.gz
./atoshid version --long        # Version: 20.3.0,commit 匹配
```

## 发布清单

- [ ] 从 `v20.x.y` tag 构建(绝不用 `v1.x`)
- [ ] `atoshid version --long` 报告的是预期版本
- [ ] Release 至少包含 `linux/amd64` + `checksums.txt`
- [ ] 链升级时:tag 版本、`app/upgrades/vX_Y/` handler、提案的 `--upgrade-name`
      三者一致
- [ ] `upgrade-info` 的二进制 URL 指向已发布资产,并带 `?checksum=sha256:…`

## 相关

- [治理操作手册](./governance-runbook.md) —— 软件升级提案
- [节点运维](./node-operation.md) —— 安装二进制、运行节点

---

*最后审阅:2026-07-22*
