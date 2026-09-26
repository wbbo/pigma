# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## 项目概述

pigma 是一个网易云音乐（及本地音频）TUI 客户端，基于 Rust + ratatui 构建。支持流式播放（边听边存）、多源 fallback（酷狗/酷我/B站/YouTube，无需 cookie）、歌词渐变高亮、CLI 控制（`pigma status` / `pigma msg`，经 IPC）与无头守护进程模式（`pigma -d`）。项目文档（README、CONTRIBUTING、SKILLS.md）均为中文。

## 常用命令

```bash
cargo build                              # 构建（Linux 需要 pkg-config libasound2-dev，rodio/ALSA 依赖）
cargo run                                # 运行 TUI（无子命令时）
cargo test --locked --all-features       # 全部测试（CI 同款）
cargo test <test_name>                   # 运行单个测试
cargo test --test ipc                    # IPC 集成测试（stub searcher，不访问网络）
cargo +nightly fmt                       # 格式化（必须 nightly，rustfmt.toml 用了 unstable 选项）
cargo +nightly fmt -- --check            # CI 的格式检查
cargo clippy                             # lint
```

首次克隆需要子模块：`git submodule update --init --recursive`。

提交信息必须遵循约定式提交（`feat:` / `fix:` / `docs:` / `refactor:` / `perf:` / `test:` / `chore:`），CHANGELOG 由 git-cliff 自动生成。main 分支受保护，需 PR 合入。CI 会跳过以 `release` 开头的提交。

发布流程（维护者）：更新 `Cargo.toml` 版本 → `git cliff --unreleased --tag x.x.x --prepend CHANGELOG.md` → `git tag vX.X.X` 并推送。

## Workspace 结构

根 `Cargo.toml` 无 `[workspace]` 段，但是隐式 workspace root；根 `Cargo.lock` 管理全部成员：

- **pigma**（根包）：TUI 应用与 CLI
- **crates/ncm-api**：网易云音乐 API 客户端（`NcmClient`、weapi 加密、cookie 登录、数据模型）
- **crates/sonar**：多源音频搜索库（kugou/kuwo/bilibili/youtube），核心是 `SonarFinder`（多 provider 搜索 + 排序）与 `SonarProvider` trait
- **crates/y7dl**：git 子模块（wbbo fork），YouTube 提取/下载库，是 sonar 的 youtube 后端

## 主 crate 架构

`src/` 采用「`foo.rs`（模块文档 + pub 重导出）+ `foo/`（实现文件）」的组织模式。`lib.rs` 暴露全部模块；`main.rs` 只保留进程级副作用（终端初始化、panic 恢复、参数分发），其余逻辑都在 lib 中，供 `cli.rs` 的 `status`/`msg` 子命令复用。

核心分层与数据流：

- **`app.rs`**：`App` 是主状态枢纽，接线 config/state/playback/service/theme/IPC，`App::run(terminal)` 是事件主循环。
- **`event.rs`**：`Event`/`AppEvent`/`AuthEvent`/`PlaybackEvent`/`NavigationEvent` 等事件类型与 mpsc 通道，是 async worker 与主渲染循环之间唯一的桥。
- **`state/`**（纯 UI 状态）与 **`ui/`**（纯渲染 widget）与 **`input/`**（按键 → 动作分发，按视图分文件：navigation/content/search/login/help/command/splash/table）三者分离。
- **`service.rs`**：`ApiService` 集中处理所有数据加载——解析 `ApiEndpoint`（导航树字符串 key → 端点）、应用缓存、把错误映射为事件。
- **`playback/`**：`PlaybackEngine`（`engine.rs`）为核心；`stream_client.rs` 流式下载（stream-download，边听边存）、`queue.rs` 播放队列、`heartbeat.rs` 心动模式、`scan.rs` 本地音乐扫描、`source.rs` 播放源 fallback。定义 `NCM_SEARCH_QUEUE_KEY` 与 `THIRD_PARTY_QUEUE_KEY` 区分队列来源。
- **`cache/`**：`CacheManager` 统一管理 audio/content/covers/lyrics/index 五类磁盘缓存。
- **`ipc.rs`**：TUI/守护进程绑定 Unix socket（`~/.cache/pigma/pigma.sock`）或 Windows 命名管道（`\\.\pipe\pigma`），协议为一行 JSON 请求 / 一行 JSON 响应：`{"cmd":"status"}`、`{"cmd":"subscribe"}`（事件推送）、`{"cmd":"msg","action":{...}}`（注入 `IpcEvent` 到 app 事件通道）。`status`/`queue` 快照通过 `Arc<Mutex<…>>` + broadcast channel 暴露给 CLI。
- **`sonar` 歌曲 id**：第三方源歌曲使用合成 id（`sonar::make_song_id` / `is_sonar_song_id`），与 NCM 数字 id 区分；搜索结果注册进 `App.search_results`，使 `pigma msg play <id>` 能播放不在队列中的搜索结果。
- **`config/`**：TOML 运行时配置（参考 `config.example.toml`），含 theme/column/navigation/playerbar 等 registry。

### 端点字符串约定

导航端点字符串（`liked`、`toplist`、`local_music` 等）在 `ApiEndpoint::parse`（src/service.rs）集中映射；部分端点有别名（如 `__liked__`）。歌单类端点解析为「一组歌单」，`ENDPOINT:N`（CLI `-d`）或 `--playlist N`（`msg switch-list`）选第 N 个，1 起始。

## 注意事项

- 终端显示依赖 Nerd Fonts（图标字符如 ``）；UI 代码中勿替换为普通字符。
- reqwest/rustls 统一使用 `rustls-no-provider` feature，进程入口需先安装 crypto provider（`rustls::crypto::ring::default_provider().install_default()`），测试同理（见 tests/ipc.rs）。
- release profile 开启 lto + codegen-units=1 + panic=abort；Linux aarch64 交叉编译由 Cross.toml 配置（需 `libasound2-dev:$CROSS_DEB_ARCH`）。
- 音频解码经 rodio + symphonia features（mp3/flac/aac/vorbis/wav），feature 列表在根 Cargo.toml 显式枚举，勿用 default-features。
