#  Git line-ending

##  遇到的问题

使用MDK-ARM建立的工程，通过git提交以后，再次使用MDK-ARM打开以后，工程文件(.uvprojx)会发生变动，对比为第一行变动，表面看不到任何不一样的地方，留意到vscode提示行尾序列EOL有变动。

## 原因

尽管使用的是windows，但是MDK-ARM打开工程文件后关闭，都会对.uvprojx文件LF结尾，好奇怪。

但是windows平台Git的默认设置会让它在check out代码自动将LF转换为CRLF，这个的原因参见下面core.autocrlf的说明。

于是两者就不统一了。

## Git是怎么处理的

以下为官方文档摘选：[Git book](https://git-scm.com/book/en/v2/Customizing-Git-Git-Configuration#Formatting-and-Whitespace)

#### `core.autocrlf`

If you’re programming on Windows and working with people who are not (or vice-versa), you’ll probably run into line-ending issues at some point. This is because Windows uses both a carriage-return character and a linefeed character for newlines in its files, whereas macOS and Linux systems use only the linefeed character. This is a subtle but incredibly annoying fact of cross-platform work; many editors on Windows silently replace existing LF-style line endings with CRLF, or insert both line-ending characters when the user hits the enter key.

Git can handle this by auto-converting CRLF line endings into LF when you add a file to the index, and vice versa when it checks out code onto your filesystem. You can turn on this functionality with the `core.autocrlf` setting. If you’re on a Windows machine, set it to `true` — this converts LF endings into CRLF when you check out code:

Git可以通过自动转换CRLF为LF来处理这种情况：当你跟踪一个文件时自动转换一次，反之，当你检出到你本地时再进行一次逆转换。

对于windows平台来讲，将CRLF转为LF进行跟踪，比如提交时全是为LF，当时检出文件时，又再次将LF转为CRLF以适应你的系统。

```console
$ git config --global core.autocrlf true
```

If you’re on a Linux or macOS system that uses LF line endings, then you don’t want Git to automatically convert them when you check out files; however, if a file with CRLF endings accidentally gets introduced, then you may want Git to fix it. You can tell Git to convert CRLF to LF on commit but not the other way around by setting `core.autocrlf` to input:

对于Linux和macOS，上边那个转换显然不合适，检出文件到本地时不需要git进行自动转换。

```console
$ git config --global core.autocrlf input
```

This setup should leave you with CRLF endings in Windows checkouts, but LF endings on macOS and Linux systems and in the repository.

通过这样的设置，windows平台的用户可以本地保持CRLF，Linux和macOS本地保持LF，不管哪一种仓库为LF。

If you’re a Windows programmer doing a Windows-only project, then you can turn off this functionality, recording the carriage returns in the repository by setting the config value to `false`:

```console
$ git config --global core.autocrlf false
```



#### `core.safecrlf`

If true, makes Git check if converting `CRLF` is reversible when end-of-line conversion is active. Git will verify if a command modifies a file in the work tree either directly or indirectly. For example, committing a file followed by checking out the same file should yield the original file in the work tree. If this is not the case for the current setting of `core.autocrlf`, Git will reject the file. The variable can be set to "warn", in which case Git will only warn about an irreversible conversion but continue the operation.

当safecrlf设置为true时，并且autocrlf激活（设置为true或者input）时，git会检查CRLF和LF互相转换的可逆性。Git 将验证是否直接或间接修改了工作树中的文件。举例来说，提交一处修改会接着检出同一文件，这个检出的文件应该和工作树中的原始文件一致。如果和autocrlf描述的情况不符，Git会排斥这个文件。safecrlf可以设置为warn，此时Git仅仅做出警告。

CRLF conversion bears a slight chance of corrupting data. When it is enabled, Git will convert CRLF to LF during commit and LF to CRLF during checkout. A file that contains a mixture of LF and CRLF before the commit cannot be recreated by Git. For text files this is the right thing to do: it corrects line endings such that we have only LF line endings in the repository. But for binary files that are accidentally classified as text the conversion can corrupt data.

CRLF转换又损坏数据的风险，一个混合了LF和CRLF的文件Git会统一它们，但是并不能恢复至原始，这对于文本文件来讲无所谓，文本文件就是需要一个统一的行尾，但是对于二进制文件，不应该进行这种转换。

If you recognize such corruption early you can easily fix it by setting the conversion type explicitly in .gitattributes. Right after committing you still have the original file in your work tree and this file is not yet corrupted. You can explicitly tell Git that this file is binary and Git will handle the file appropriately.

.gitattributes文件应该尽早设置以方便Git可以知道那些是二进制文件以便适当的处理它们。

Unfortunately, the desired effect of cleaning up text files with mixed line endings and the undesired effect of corrupting binary files cannot be distinguished. In both cases CRLFs are removed in an irreversible way. For text files this is the right thing to do because CRLFs are line endings, while for binary files converting CRLFs corrupts data.

Note, this safety check does not mean that a checkout will generate a file identical to the original file for a different setting of `core.eol` and `core.autocrlf`, but only for the current one. For example, a text file with `LF` would be accepted with `core.eol=lf` and could later be checked out with `core.eol=crlf`, in which case the resulting file would contain `CRLF`, although the original file contained `LF`. However, in both work trees the line endings would be consistent, that is either all `LF` or all `CRLF`, but never mixed. A file with mixed line endings would be reported by the `core.safecrlf` mechanism.

不会允许混合CRLF和LF的文件存在，会报错。

## 添加.gitattributes

配置 *.gitattribute* 文件来管理 Git 如何处理特定仓库中的行结束符，将此文件提交到仓库时，它将覆盖所有仓库贡献者的 `core.autocrlf` 设置。 这可确保所有用户的行为一致，而不管其 Git 设置和环境如何。

*.gitattributes* 文件必须在仓库的根目录下创建，且像任何其他文件一样提交。

*.gitattributes* 文件看上去像一个两列表格。

- 左侧是 Git 要匹配的文件名。
- 右侧是 Git 应对这些文件使用的行结束符配置。

一个类型的文件有三种设置：`text`, `text eol=crlf`, `binary`。 我们将在下面介绍一些可能的设置。

- `text=auto` Git 将以其认为最佳的方式处理文件。 这是一个合适的默认选项。
- `text eol=crlf` 在检出时 Git 将始终把行结束符转换为 `CRLF`，将其用于必须保持 `CRLF` 结束符的文件，即使在 OSX 或 Linux 上。
- `text eol=lf` 在检出时 Git 将始终把行结束符转换为 `LF`。 您应将其用于必须保持 LF 结束符的文件，即使在 Windows 上。
- `binary` Git 会理解指定文件不是文本，并且不应尝试更改这些文件。 `binary` 设置也是 `-text -diff` 的一个别名。
