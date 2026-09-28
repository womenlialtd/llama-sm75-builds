# llama-sm75-builds

**个人用的 Windows + CUDA sm_75 (GTX 1660) 自动构建仓库。**

这个仓库**不含 llama.cpp 源码**，只有一个 GitHub Actions workflow。构建时它去检出上游 [`ggml-org/llama.cpp`](https://github.com/ggml-org/llama.cpp) 的指定 tag，原样编译（**不打任何补丁**），然后把产物发到本仓库的 Releases。

源码与功能说明请看上游：[ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp)

---

## 为什么这么设计

一开始我是 fork 上游来构建的（见下方"相关仓库"），但 fork 有两个持续成本：

- 上游自带的 workflow 会在我的仓库里**空跑**（`Code Style Checker`、`EditorConfig Checker`、`build-cann.yml`），白烧 Windows runner 配额；
- 跟进新版本要 sync / merge，容易和上游改动纠缠。

改成纯 workflow 仓库后，升级只需要改一行 `UPSTREAM_REF`，且不会有别人的 workflow 混进来。

---

## 怎么用

手动触发 workflow `llama.cpp Windows CUDA sm_75 Build`（`workflow_dispatch`），约 30–35 分钟。产物自动出现在 Releases。

跟进上游新版本：改 `.github/workflows/sm75-win-cuda.yml` 里这三处即可 ——

```yaml
UPSTREAM_REPO: ggml-org/llama.cpp
UPSTREAM_REF:  b11213      # ← 换 tag（用具体 tag，不要用 master）
BUILD_LABEL:   sm75-llama-b11213
ZIP_NAME:      llama-b11213-bin-win-cuda13.4-sm75-x64.zip
```

换 tag 前建议先确认新 tag 的 CMake 选项仍齐全（上游会增删选项，传未知 `-D` 会让 configure 直接失败，不是忽略警告）。

---

## 当前产物

| Release tag | 源码 | 说明 |
|---|---|---|
| `sm75-llama-b11213` | 上游 `b11213`，无补丁 | `llama-b11213-bin-win-cuda13.4-sm75-x64.zip`，45 文件 / 53,000,214 B（50.5 MiB） |

编译配置：`windows-2022` + MSVC(VS2022) + Ninja Multi-Config，CUDA **13.4.1 GA**（走 NVIDIA redist 归档免安装部署），关键选项

```
-DGGML_NATIVE=OFF -DGGML_BACKEND_DL=ON -DGGML_CPU_ALL_VARIANTS=ON
-DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=75
```

具体的 sha256、体积等以各 Release 页面正文为准（同配置重跑字节会变，哈希也会变）。

---

## 这个包和官方包的差别

| | 官方 `cuda-13.4-x64` 包 | 本仓库的包 |
|---|---|---|
| sm_75 机器码 | **只有 PTX**，运行期由驱动 JIT | **原生 SASS**（只编 arch 75） |
| `ggml-cuda.dll` | 140 MiB（多架构 fatbin） | 52 MiB（单架构） |
| 源码改动 | — | 无，上游原样 |

架构差异是解包 `.nv_fatbin` 的 zstd 帧、读 cubin `e_flags` 实测出来的：官方包真 SASS 集合是 `{86, 89, 120, 121}`，本包是 `{75}`。根因在上游默认架构串里 `75` 写作 `75-virtual`（只出 PTX）。

所以这个包的意义是**免去驱动 JIT 的首次加载开销 + 包体小得多**；稳态解码速度提升有限（JIT 产物与 nvcc 走同一后端），想要确定数值请自己跑 `llama-bench` 对比。

## 两个必须知道的代价

1. **换卡即失效**。只编了 sm_75，插非 Turing 显卡时 CUDA 后端没有对应 kernel，会**静默回落到 CPU**（不报错）。
2. **不含 CUDA 运行时**。`ggml-cuda.dll` 需要 `cublas64_13.dll`，本包与官方包一样都不附带 —— 需要已装 CUDA 13.x，或另取上游的 `cudart-llama-bin-win-cuda-13.4-x64.zip`。

验收时请看运行日志出现 `load_backend: loaded CUDA backend`：因为构建开了 `GGML_BACKEND_DL=ON`，后端 DLL 缺失只在运行期静默降级，**zip 解压完整 ≠ 真的跑在 GPU 上**。

---

## 相关仓库

[`womenlialtd/k2-llama-sm75`](https://github.com/womenlialtd/k2-llama-sm75) —— fork 形态，构建**含 K2-Horizon 模型支持**的分支。本仓库编的是官方上游、**不含 K2-Horizon**，两者不是互相升级的关系：要 K2 模型用那个，要跟进上游最新版用这个。

---

*个人自用构建仓库。上游 llama.cpp 的版权与维护归 ggml-org。*
