# 客户机文件系统

Sogen 给客体的文件系统是一张挂载表。每个挂载把一个客体路径（比如 `C:\` 或 `C:\out`）连接到一个宿主机文件或目录，并带有一种模式：`ro`（只读）、`rw`（读写），或者仅用于根驱动器的 `cow`（写时复制，见下文）。要查找某个客体路径对应的挂载，Sogen 会使用包含该路径的最深的挂载点。因此 `C:\out` 挂载会覆盖 `C:\` 驱动器对所有位于 `C:\out` 之下的路径。

带有模拟根目录时，`<root>/filesys` 中每个单字母目录都会成为一个驱动器挂载。例如，`<root>/filesys/c` 会成为 `C:\`。这些驱动器使用 `--root-mode` 指定的模式，默认为 `ro`。没有根目录时（仅限 Windows 宿主机），Sogen 直接挂载宿主机驱动器。注册表（`<root>/registry`）和 DPAPI 文件（`<root>/dpapi`）位于 `filesys` 之外，因此客体无法打开它们。Sogen 会从根目录读取 DPAPI 主密钥，但绝不会写入它。

沙箱还可以用写时复制方式挂载根驱动器：`--root-mode cow --root-layer DIR`。该模式需要一个模拟根目录和一个图层目录，并且只对根驱动器生效。客体对驱动器 `C:\` 的写入会进入 `DIR/c/upper`，删除操作会记录在 `DIR/c/state` 中。图层目录使用小写的驱动器字母。根目录本身不会改变。子进程使用同一图层，之后用同一图层再运行一次可以看到先前的更改。Android 应用就是每个游戏使用一个图层。

用 `--mount HOST:GUEST[:MODE]` 增加挂载。HOST 可以是文件或目录。MODE 为 `ro` 或 `rw`。如果不给 MODE，挂载使用默认模式 `ro`。`-p GUEST HOST` 等同于 `--mount HOST:GUEST`。在 Python 中使用 `root_mode="ro"` 和 `mounts=[(host, guest)]` 或 `mounts=[(host, guest, mode)]`。

客体对 `ro` 挂载的写入会失败并返回 `STATUS_ACCESS_DENIED`，宿主机文件不会有任何变化。这适用于：

- 以写权限打开文件，
- 创建文件或目录，
- 删除或重命名（源和目标都必须可写，即 `rw` 或 `cow`），
- 除 `FilePositionInformation` 之外的所有 `NtSetInformationFile` 类别（例如时间、属性、大小和 close 时删除），
- 对文件句柄的 `NtSetSecurityObject`。

有一些规则适用于所有挂载。客体不能删除或重命名目录挂载的根，但可以删除或重命名文件挂载。跨驱动器的重命名会失败并返回 `STATUS_NOT_SAME_DEVICE`。Sogen 通过文本比较路径来检查客体路径是否仍留在其宿主机目录内。它不跟随符号链接，所以宿主机上的符号链接可以指向挂载之外。

`--mount` 解析器从右侧按冒号拆分规格。它从最后一个冒号读取 MODE，GUEST 从驱动器字母开始。因此 HOST 可以包含冒号，比如 Windows 宿主机上的 `C:\data`。因为 MODE 取自最后一个冒号，GUEST 不能命名 NTFS 备用数据流。`--mount .\a.txt:C:\a.txt:ro` 是 `C:\a.txt` 的只读挂载，而不是名为 `ro` 的数据流的挂载。其他流名，比如 `C:\a.txt:data`，会被拒绝并报错。我们认为没有人会挂载数据流，所以对于简短的 CLI 语法来说这个限制是可以接受的。该限制只适用于 CLI，因为 Python 中的挂载是元组。

存档状态（savestate）不包含挂载表。加载存档状态时，请再次使用相同的挂载。如果存档中的某个文件句柄需要写访问（写句柄、close 时删除或挂起的重命名），而其挂载现在是 `ro`，加载会失败并报出该文件名的错误。
