# UEFIReader

从 **Qualcomm 风格 UEFI / XBL 整包镜像** 中解出固件卷（Firmware Volume / FFS）里的各段数据，并生成可在 **EDK2 / 其它 UEFI 工程** 中引用的 **`.inf` 描述**及辅助列表（`DXE.inc`、`APRIORI.inc` 等）。

> 原实现思路与版权说明见 `UEFIReader/UEFI.cs` 等文件内注释；核心逻辑为解析 PI 固件卷、解 GUID 定义段中的 **LZMA / Gzip** 压缩体，并导出 `PE32`（以 `.efi` 保存）、`DXE_DEPEX` 等节。

## 功能概览

| 项目 | 说明 |
|------|------|
| **定位卷头** | 在镜像中搜索 ASCII 标记 `_FVH`，以此前 **0x28** 为 Volume 起点（前 **0x28** 为 Qualcomm 分区头，随后为 EDK2 PI 固件卷，见源码注释） |
| **解析结构** | 解析 FV、FFS 文件项（DRIVER、APPLICATION、DXE_CORE、嵌套 FV 等）与 Section（PE32、UI、DEPEX、RAW、GUID 定义段等） |
| **解压缩** | 对 `EFI_SECTION_GUID_DEFINED` 中已支持的算法解压：`LZMA`、`Gzip`（见 `UEFI.ParseGuidDefinedSection`） |
| **还原构建路径** | 从已解压的 PE/数据里用正则扫出 `*.dll` 路径，尽量恢复类似源码树子目录，用于输出目录和 `.inf` 命名 |
| **构建 ID** | 若映像里存在 `QC_IMAGE_VERSION_STRING=...` 模式，会取为 **BuildId**；有值时，输出会落在 **`<输出目录>/<BuildId>/`** 下 |
| **DXE 列表** | 为带路径/模块的项生成每模块的 `*.inf`，并写出 `DXE.dsc.inc`、`DXE.inc`；根据 **DXE apriori** 自由格式卷解析出的 GUID 列表，生成 `APRIORI.inc` |
| **无路径资源** | 无 UI/路径的 RAW 等会落到 **`RawFiles/`** 等约定目录 |

**不适用**：非上述布局的 UEFI 映像、未实现的 GUID 压缩类型、或校验无法通过的损坏镜像；此时可能抛出 `BadImageFormatException` 等异常结束。

## 使用方法

**命令行**（两个参数，顺序固定）：

```text
UEFIReader  <UEFI 或 XBL 镜像文件路径>  <输出目录>
```

- **参数 1**：要解析的**单一文件**（如从设备/ROM 中导出的全镜像或分区分卷导出文件，需存在于磁盘上）。
- **参数 2**：**输出根目录**；若成功解析到 BuildId，实际写入路径为 `输出目录/BuildId/`（若 `BuildId` 为空，则直接写入 `输出目录`）。

**示例**（Windows，根据实际路径修改）：

```bat
UEFIReader.exe  D:\firmware\uefi_xbl.bin  D:\out\extracted
```

若 `QC_IMAGE_VERSION_STRING` 为 `QcomPkg/...` 之类，结果常在 `D:\out\extracted\该版本字符串\` 下。

## 输出内容说明

在**最终输出目录**中（含 BuildId 子目录时，指该子目录内），典型包括：

- **各模块子目录**下的 `*.inf` 及对应二进制（如 **`*.efi`** 来自 `PE32` 节、**`*.depex`** 来自 `DXE_DEPEX` 等）；
- **`DXE.dsc.inc`**：供 DSC 的 `!include` 等使用的路径列表（项目内为每行一个 `.inf` 相对风格路径）；
- **`DXE.inc`**：FDF/片段风格列表，含 `INF` 行或 `FILE FREEFORM` 等片段；
- **`APRIORI.inc`**：在解析到 apriori 列表时，为需要优先加载的项生成 `APRIORI DXE { ... }` 块；
- **`RawFiles\`**：部分 RAW/自由形式资源的落盘位置。

## 从源码构建

- **环境**：.NET 8 SDK（`TargetFramework: net8.0`）
- 在解决方案目录执行：

```bat
dotnet restore
dotnet build UEFIReader.sln -c Release
```

可执行文件在 `UEFIReader\bin\Release\net8.0\`；跨平台需自行 `dotnet publish -r <RID>`。CI 中若多 RID 发布，项目内已配置 `RuntimeIdentifiers` 以配合 `restore`。

## 代码结构（简要）

| 文件 / 区域 | 作用 |
|-------------|------|
| `Program.cs` | 入口：读入镜像字节，构造 `UEFI`，调用 `ExtractUEFI` |
| `UEFI.cs` | 固件卷/FFS/Section 遍历、校验、解 GUID 节、写 `.inc`/`.inf`/`RawFiles` |
| `LZMA.cs` / `GZip.cs` / `7zip/...` | 压缩流解压与 7z/LZ 相关实现 |
| `ByteOperations.cs` | 小端读写字节、GUID、对齐、字符串查找等 |

## 许可与致谢

- 整体仓库以根目录 [LICENSE](LICENSE) 为准；部分源文件另含 **MIT** 等声明时，以该文件头为准。
- `UEFI.cs` 等含 **Rene Lergner (@Heathcliff74xda)** 版权与许可条款的，保留其声明。
