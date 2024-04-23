---
title: Linux kernel代码提交和patch使用
date: 2023-03-09 15:19:37
tags: linux
categories: linux

---

本文描述：如何参与到Linux kernel社区中，为Linux kernel提交Patch代码；以Linux子系统MMC/SD为例介绍如何使用patch。

## Linux kernel提交代码的基本概念

### 如何参与Linux内核开发

Linux kernel的官方网站：[kernel.org](https://kernel.org/)

kernel.org内的中文文档：[如何参与Linux内核开发](https://www.kernel.org/doc/html/latest/translations/zh_CN/process/howto.html), 其中最常用的：

- 内核源码库：https://elixir.bootlin.com/ 在线查看kernel源码而无需git下载
- 内核子系统(subsystem)的补丁(patch)列表：https://patchwork.kernel.org/ 显示正在发布、评论或修订的patch： 
- 内核邮件列表的存档(archive)：https://lore.kernel.org/lkml/ 所有正在进行或已存档的patchwork都能在此找到邮件记录：

### 如何提交Patch

Patch是提交到kernel之前的一个阶段，由kernel subsystem maintainer review后**有机会**进入Linux kernel Mainline。事实上绝大所述patch最终未进入Linux kernel Mainline，仅存档到了邮件列表，在lore/patchwork.kernel.org可查看这部分patch的内容和提交过程。

- 提交Patch的总体规范参考：

  [提交补丁：如何让你的改动进入内核](https://docs.kernel.org/translations/zh_CN/process/submitting-patches.html)

- 具体地讲如何向kernel提交patch和使用patch（需要详细看）: 

  [Submitting patches: the essential guide to getting your code into the kernel](https://www.kernel.org/doc/html/v4.11/process/submitting-patches.html)

  [Applying Patches To The Linux Kernel](https://www.kernel.org/doc/html/v4.11/process/applying-patches.html?highlight=applying%20patches%20linux%20kernel)

- 关于patch命令如何使用，参考： 

  [patch-command-examples](https://www.thegeekstuff.com/2014/12/patch-command-examples/)

  [patch(1) — Linux manual page](https://www.man7.org/linux/man-pages/man1/patch.1.html) 

  [Linux下生成patch和打patch](https://blog.csdn.net/dl0914791011/article/details/17299103)

## 示例：Linux MMC子系统中UHS-II Patch的演化过程

### Linux MMC子系统的现状

MMC子系统主要包含SD card, eMMC card, SDIO几部分，Kernel Mainline的支持情况参考：[SD/eMMC: new speed modes and their support in Linux](https://elinux.org/images/9/91/Clement-sd-mmc-high-speed-support-in-linux-kernel_0.pdf#:~:text=%E2%96%B6New%20speed%20modes%20%28name%20are%20base%20on%20the,the%203.3V%20forDS%28Default%20Speed25MHz%29%20andHS%28High%20Speed%20at%2050MHz%29)

这里只关注SD card, Kernel Mainline在当前时间点（kernel 6.2）：

- 不支持UHS-II (SD 4.0 specification)
- SD express(SD 7.0 specification)在Kernel 5.11版本以后是支持的
- SD UHS-I (SD 3.0 specification)和更老版本的SD协议则在kernel 3.0就已经支持

### Linux MMC UHS-II patch的演变

Linux MMC子系统的维护者可以在[patchwork.kernel.org](https://patchwork.kernel.org/)的MMC development的about页面看到：

![image-20230309163541697](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202303091635752.png)

在patch页面可以搜索以[UHS-II为关键字的相关patch](https://patchwork.kernel.org/project/linux-mmc/list/?q=UHS-II&archive=both&series=&submitter=&delegate=&state=*)

![image-20230309162128061](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202303091621148.png)结果如下：

![image-20230309162327522](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202303091623615.png)

![](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202303091623615.png) 

具体看一下上面这些UHS-II patch的内容和reviewer的评论：

1.首次提交是[Intel的yisun](https://patchwork.kernel.org/project/linux-mmc/patch/1419672479-30852-2-git-send-email-yi.y.sun@intel.com/), 该patch被MMC维护者Ulf Hansson评论：应该split it up，之后就没有再修改和提交。

```
[RFC,1/2] mmc: core: support UHS-II in core stack.

Commit Message

[yisun1](https://patchwork.kernel.org/project/linux-mmc/list/?submitter=102631)Dec. 27, 2014, 9:27 a.m. UTC

This patch adds the UHS-II support in core layer. This is a RFC patch for
community review.

Signed-off-by: Yi Sun <yi.y.sun@intel.com>
---
 drivers/mmc/core/Makefile |    3 +-
 drivers/mmc/core/bus.c    |    5 +-
 drivers/mmc/core/core.c   |   89 ++++-
 drivers/mmc/core/sd.c     |   15 +
 drivers/mmc/core/sd_ops.c |   12 +
 drivers/mmc/core/uhs2.c   |  908 +++++++++++++++++++++++++++++++++++++++++++++
 drivers/mmc/core/uhs2.h   |   26 ++
 include/linux/mmc/core.h  |    6 +
 include/linux/mmc/host.h  |   27 ++
 include/linux/mmc/uhs2.h  |  274 ++++++++++++++
 10 files changed, 1356 insertions(+), 9 deletions(-)
 create mode 100644 drivers/mmc/core/uhs2.c
 create mode 100644 drivers/mmc/core/uhs2.h
 create mode 100644 include/linux/mmc/uhs2.h

Comments

[Ulf Hansson](https://patchwork.kernel.org/project/linux-mmc/list/?submitter=45281)Jan. 21, 2015, 10:31 a.m. UTC | [#1](https://patchwork.kernel.org/comment/12007791/)

Even if this an RFC, me and likely everybody else just stops from
reviewing this patch by looking at the above change log.

Is there a way to split it up?

Kind regards
Uffe
```

2. Genesys的Ben Chuang, Jason Lai, Victor.shih 和linaro 的akashi 在Intel的UHS-II patch上不断提交修改后的UHS-II patch（V3~V6）跟随着Kernel版本不断演化，此patch完整内容可在GitLab查看 [linux-uhs2-gl9755](https://gitlab.com/ben.chuang/linux-uhs2-gl9755)，在patchwork也可以查看commit内容和review意见：[V6 patch的第6/24提交](https://patchwork.kernel.org/project/linux-mmc/patch/20221213090047.3805-7-victor.shih@genesyslogic.com.tw/)：

```
[V6,06/24] mmc: core: Support UHS-II card control and access

Commit Message 

[Victor Shih](https://patchwork.kernel.org/project/linux-mmc/list/?submitter=207469) Dec. 13, 2022, 9 a.m. UTC

Embed UHS-II access/control functionality into the MMC request
processing flow.

Signed-off-by: Ulf Hansson <ulf.hansson@linaro.org>
Signed-off-by: Jason Lai <jason.lai@genesyslogic.com.tw>
Signed-off-by: Victor Shih <victor.shih@genesyslogic.com.tw>
---
 drivers/mmc/core/block.c   |    6 +-
 drivers/mmc/core/core.c    |   20 +
 drivers/mmc/core/mmc_ops.c |   25 +-
 drivers/mmc/core/mmc_ops.h |    1 +
 drivers/mmc/core/sd.c      |   11 +-
 drivers/mmc/core/sd.h      |    3 +
 drivers/mmc/core/sd_ops.c  |   13 +
 drivers/mmc/core/sd_ops.h  |    3 +
 drivers/mmc/core/sd_uhs2.c | 1171 +++++++++++++++++++++++++++++++++++-
 9 files changed, 1206 insertions(+), 47 deletions(-)

Comments

[Adrian Hunter](https://patchwork.kernel.org/project/linux-mmc/list/?submitter=31052) Jan. 5, 2023, 9:26 p.m. UTC | [#1](https://patchwork.kernel.org/comment/25148889/)

> +u32 sd_uhs2_select_voltage(struct mmc_host *host, u32 ocr)
> +{
...
> +
> +	if (host->caps2 & MMC_CAP2_FULL_PWR_CYCLE) {
> +		bit = ffs(ocr) - 1;
> +		ocr &= 3 << bit;
> +		/* Power cycle */
> +		err = sd_uhs2_power_off(host);
> +		if (err)
> +			return 0;
> +		err = sd_uhs2_reinit(host);

This looks circular:

sd_uhs2_select_voltage
-> sd_uhs2_reinit
   -> sd_uhs2_init_card
      -> sd_uhs2_legacy_init
         -> sd_uhs2_select_voltage

[Ulf Hansson](https://patchwork.kernel.org/project/linux-mmc/list/?submitter=45281) Feb. 8, 2023, 3:30 p.m. UTC | [#2](https://patchwork.kernel.org/comment/25202573/)

> diff --git a/drivers/mmc/core/block.c b/drivers/mmc/core/block.c
> index 20da7ed43e6d..d3e8ec43cdd5 100644
> --- a/drivers/mmc/core/block.c
> +++ b/drivers/mmc/core/block.c
> @@ -1596,6 +1596,9 @@ static void mmc_blk_rw_rq_prep(struct mmc_queue_req *mqrq,
>         struct request *req = mmc_queue_req_to_req(mqrq);
>         struct mmc_blk_data *md = mq->blkdata;
>         bool do_rel_wr, do_data_tag;
> +       bool do_multi;
> +
> +       do_multi = (card->uhs2_state & MMC_UHS2_INITIALIZED) ? true : false;
>
>         mmc_blk_data_prep(mq, mqrq, recovery_mode, &do_rel_wr, &do_data_tag);
>
> @@ -1606,7 +1609,7 @@ static void mmc_blk_rw_rq_prep(struct mmc_queue_req *mqrq,
>                 brq->cmd.arg <<= 9;
>         brq->cmd.flags = MMC_RSP_SPI_R1 | MMC_RSP_R1 | MMC_CMD_ADTC;
>
> -       if (brq->data.blocks > 1 || do_rel_wr) {
> +       if (brq->data.blocks > 1 || do_rel_wr || do_multi) {

This looks wrong to me. UHS2 can use single block read/writes too. Right?

>                 /* SPI multiblock writes terminate using a special
>                  * token, not a STOP_TRANSMISSION request.
>                  */
> @@ -1619,6 +1622,7 @@ static void mmc_blk_rw_rq_prep(struct mmc_queue_req *mqrq,
>                 brq->mrq.stop = NULL;
>                 readcmd = MMC_READ_SINGLE_BLOCK;
>                 writecmd = MMC_WRITE_BLOCK;
> +               brq->cmd.uhs2_tmode0_flag = 1;

As "do_multi" is always set for UHS2, setting this flag here seems to
be wrong/redundant.

Anyway, if I understand correctly, the flag is intended to be used to
inform the host driver whether the so-called 2L_HD_mode (half-duplex
or full-duplex) should be used for the I/O request or not.

To fix the above behaviour, I suggest we try to move the entire
control of the flag into mmc_uhs2_prepare_cmd(). We want the flag to
be set for multi block read/writes (CMD18 and CMD25), but only if the
host and card supports the 2L_HD_mode too. According to my earlier
suggestions, we should be able to check that via the bits we set
earlier in the ios->timing.

Moreover, by making mmc_uhs2_prepare_cmd() responsible for setting the
flag, I think we can also move the definition of the flag into the
struct uhs2_command. While at it, I suggest we also rename the flag
into "tmode_half_duplex", to better describe its purpose, which also
means the interpretation of the flag becomes inverted.
```

## 详解Patch的使用

Kernel document: [Applying Patches To The Linux Kernel](https://www.kernel.org/doc/html/latest/process/applying-patches.html#:~:text=A%20patch%20is%20a%20small%20text%20document%20containing,the%20patch%20will%20change%20the%20source%20tree%20into.)

### Patch与git diff

Patch文件的内容实际是`git diff`命令的输出，git diff的输出定义为.diff文件或.patch文件，即可作为patch使用。打patch实际上就是按diff规则，解析diff/patch文件，去改变本地的代码树和内容。

git diff说明文档参考 [git-diff](https://git-scm.com/docs/git-diff)，比较常用的是使用`git diff [<path>…]`输出某个路径/文件的差异；如果path为空，则输出当前git仓库所有文件的差异。

如下示例：在drivers/mmc/core/block.c增加修改了`//AAAAAAAAA`，在drivers/mmc/core/block.h增加了`//BBBBBBBBB`，以下详细说明git diff 输出的含义：

- diff --git a/drivers/mmc/core/block.c b/drivers/mmc/core/block.c：用git diff命令，比较a和b版本的drivers/mmc/core/block.c，a和b是diff用来区分同名文件的标识，不是实际路径。
- index 7fa83e5..8963e57：这个diff如果被commit提交，commit-id将是index值7fa83e5..8963e57。
- --- a/drivers/mmc/core/block.c 和+++ b/drivers/mmc/core/block.c 同时存在：表示是对已存在的block.c文件有内容修改；与之相对的是某个文件只有+++或---，表示是新增文件文件，或者是删除了文件。
- @@ -1829,6 +1829,8 @@ static void mmc_blk_mq_rw_recovery(struct mmc_queue *mq, struct request *req)：该修改代码所在的行数以及所在的函数名。
- +//AAAAAAAAA：具体的修改内容，+是新增，-是删除。

```
diff --git a/drivers/mmc/core/block.c b/drivers/mmc/core/block.c
index 7fa83e5..8963e57 100644
--- a/drivers/mmc/core/block.c
+++ b/drivers/mmc/core/block.c
@@ -1829,6 +1829,8 @@ static void mmc_blk_mq_rw_recovery(struct mmc_queue *mq, struct request *req)
        u32 blocks;
        int err;

+//AAAAAAAAA

/*

diff --git a/drivers/mmc/core/block.h b/drivers/mmc/core/block.h
index 31153f6..5501895 100644
--- a/drivers/mmc/core/block.h
+++ b/drivers/mmc/core/block.h
@@ -17,4 +17,6 @@ struct work_struct;

 void mmc_blk_mq_complete_work(struct work_struct *work);

+//BBBBBBBBB

 #endif
```

如果是已经git commit的两个版本之间的diff, 可直接产生所有修改内容的diff文件:

```
git diff commit-a commit-b
```

一般提交给Kernel社区的patch需要按功能和文件拆分成多个patch提交，也就是说应该对某个文件或者路径git diff, 而不建议直接对版本所有文件git diff。例如以上patch可以分为两个diff，内容等价于：

```
git diff drivers/mmc/core/block.c

git diff drivers/mmc/core/block.h
```

### Patch与kernel版本

为了正确打一个补丁，你需要知道这个补丁是从哪个基础代码版本(base)产生的，以及这个补丁会使源码树升级成哪个版本。

#### 用于Kernel升级的官方patch

在kernel.org可以看到有很多Kernel版本之间有patch可以用于升级kernel，例如从kernel 4.19.275升级到5.4.234，可以下载并安装patch-5.4.234.xz

![image-20230309194230288](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202303091942357.png)

#### patchwork社区的第三方patch 

可以下载diff或者series去获取patch文件，根据patch提交时间和代码上下文大致估计当时的Kernel版本

- diff: 当前patch的diff, 由于一个大patch可能被拆分为多个小patch，此文件通常为某个小patch
- mbox: 在diff基础上包含了邮件信息（MIME信息）
- series: 整个功能的所有patch系列的mbox合并内容，包括邮件信息（MIME信息）

![image-20230309194559245](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202303091945285.png)

Patch命令使用以上的.diff文件，有的也命名为.patch文件

```
patch -p1 < xxx.diff
```

-p 表示path：跳过第几级目录；1 表示忽略第一级目录

例如diff如下时，第一级目录用a, b表示，patch -p1将忽略a, b，将drivers/mmc/xxx的diff内容打patch到当前：

```
diff --git a/drivers/mmc/core/bus.c b/drivers/mmc/core/bus.c
index 86d2711..6565754 100644
--- a/drivers/mmc/core/bus.c
+++ b/drivers/mmc/core/bus.c
@@ -308,8 +308,9 @@ int mmc_add_card(struct mmc_card *card)
 	} else {
 		pr_info("%s: new %s%s%s%s%s card at address %04x\n",
 			mmc_hostname(card->host),
-			mmc_card_uhs(card) ? "ultra high speed " :
-			(mmc_card_hs(card) ? "high speed " : ""),
+			mmc_card_uhs2(card) ? "ultra high speed 2 " :
+			(mmc_card_uhs(card) ? "ultra high speed 1" :
+			(mmc_card_hs(card) ? "high speed " : "")),
 			mmc_card_hs400(card) ? "HS400 " :
 			(mmc_card_hs200(card) ? "HS200 " : ""),
 			mmc_card_ddr52(card) ? "DDR " : "",
```

对于一个大功能的多个patch series，需要分别下载各diff文件； 或者一次下载series后手动删除所有MIME信息。

#### 如何寻找Patch对应的kernel版本

如果Patch和kernel版本不匹配，patch命令无法合并patch到此kernel中，导致patch失败，因此打patch首先要确定其对应哪个kernel版本。

（1）如果patch commit是已提交到kernel的官方patch，则可以根据commit-id查找包含此commit的kernel版本，参考：[Finding a patch's kernel version with git](https://lwn.net/Articles/392293/)

```
git describe --contains <commit-id>
```

（2）大多数patch是没提交到kernel的第三方patch，因此patch中的index在kernel是找不到的，所以只能通过提交邮件的信息确定适用的kernel版本。

以前文提到的 [RFC PATCH v3.1 16/27](https://lore.kernel.org/all/20201106022726.19831-2-takahiro.akashi@linaro.org/T/#u)为例，patch是在提交时间点的kernel master版本或tag版本上测试的。

> ```
> [auto build test WARNING on linus/master]
> [also build test WARNING on v5.10-rc2]
> [cannot apply to v3.1 next-20201105]
> [If your patch is applied to the wrong git tree, kindly drop us a note
> ```

另外一个示例：提交者在提交信息中写了基于哪个kernel版本：[Add support UHS-II for GL9755](https://lore.kernel.org/all/20221213090047.3805-24-victor.shih@genesyslogic.com.tw/T/#u)

> ```
> Changes in v6 (Dec. 12, 2022)
> * rebased to the linux-kernel-v6.1.0-rc8 in Ulf Hansson next branch.
> 
> Changes in v5 (Oct. 19, 2022)
> * rebased to the linux-kernel-v6.1-rc1 in Ulf Hansson next branch.
> ```

如果一个第三方patch没有任何kernel版本的信息，只能通过提交时间来尝试kernel，一般情况下不建议这种尝试，因为提交者使用的可能是当时最新的kernel, 也可能是一两个月前的kernel, 中间可能有很多-rc版本。

下面以[RFC 0/2 mmc: UHS-II implementation](https://lore.kernel.org/all/525EAED47491124EB5123A51BD2FC79101A30EE2@SHSMSX101.ccr.corp.intel.com/)为例，尝试寻找此patch可应用的kernel版本，此patch提交信息如下：

```
* RE: [RFC 0/2] mmc: UHS-II implementation
  2014-12-27  9:27 [RFC 0/2] mmc: UHS-II implementation Yi Sun
  2014-12-27  9:27 ` [RFC 1/2] mmc: core: support UHS-II in core stack Yi Sun
  2014-12-27  9:27 ` [RFC 2/2] mmc: sdhci: support UHS-II in SDHCI host Yi Sun
```

（1）首先在linux kernel git tag时间记录找到接近此patch提交时间的kernel版本：

在[linux kernel github](https://github.com/torvalds/linux) 下拉tag列表，找接近patch申请时间的kernel release版本，可见kernel version < 4.0是此patch可能适用的版本

![image-20230313113324650](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202303131133722.png)

（2）patch内容的函数名和上下文

```
@@ -248,6 +252,12 @@ mmc_start_request(struct mmc_host *host, struct mmc_request *mrq)
 			mrq->stop->mrq = mrq;
 		}
 	}
+
+	if (host->flags & MMC_UHS2_SUPPORT &&
+	    host->flags & MMC_UHS2_INITIALIZED)
+		if (mrq->cmd->uhs2_cmd == NULL)
+			uhs2_prepare_sd_cmd(host, mrq);
+
 	mmc_host_clk_hold(host);
 	led_trigger_event(host->led, LED_FULL);
 	host->ops->request(host, mrq);
```

（3）在 [bootlin](https://elixir.bootlin.com/linux/v4.9/source/drivers/mmc/core/core.c#L264) 找到kernel的同函数并对比上下文：

```
	if (mrq->data) {
       ...
       
		mrq->cmd->data = mrq->data;
		mrq->data->error = 0;
		mrq->data->mrq = mrq;
		if (mrq->stop) {
			mrq->data->stop = mrq->stop;
			mrq->stop->error = 0;
			mrq->stop->mrq = mrq;
		}
	}
	
	///// 此处为patch添加处
	
	led_trigger_event(host->led, LED_FULL);
	__mmc_start_request(host, mrq);

	return 0;
```

在stable kernel版本上尝试此patch (跳过-rc版本)，首先找kernel tag早于此patch邮件的时间，尝试了kernel 3.18, 3.17都有patch fail，如下可见patch和kernel有少量代码offset能自动匹配，但是有些差异patch搞不定，例如有merge代码冲突会导致对应的Hunk # FAILED，hunk是patch中的diff --git的代码块。

```
ubuntu@ubuntu-Z390-GAMING-X:~/linux-4.9$ patch -p1 < RFC-1-2-mmc-core-support-UHS-II-in-core-stack..diff 
patching file drivers/mmc/core/Makefile
Hunk #1 FAILED at 7.
1 out of 1 hunk FAILED -- saving rejects to file drivers/mmc/core/Makefile.rej
patching file drivers/mmc/core/bus.c
Hunk #1 succeeded at 334 with fuzz 2 (offset 26 lines).
patching file drivers/mmc/core/core.c
Hunk #2 FAILED at 36.
Hunk #3 succeeded at 63 with fuzz 2 (offset 6 lines).
Hunk #4 FAILED at 250.
Hunk #5 succeeded at 503 (offset 116 lines).
Hunk #6 succeeded at 518 (offset 116 lines).
Hunk #7 FAILED at 425.
...
6 out of 17 hunks FAILED -- saving rejects to file drivers/mmc/core/core.c.rej
```

在kernel 3.18打此patch，只有一个fail，可以根据此fail进一步定位

```
Hunk #13 FAILED at 2339.
1 out of 17 hunks FAILED -- saving rejects to file drivers/mmc/core/core.c.rej
```

core.c.rej 内容如下，注意这里的行号是已经经过patch操作被偏移的代码的行号，实际行号应该去patch原文件查看此hunk的行号，这里只看是什么函数名。

```
--- drivers/mmc/core/core.c
+++ drivers/mmc/core/core.c
@@ -2339,7 +2391,9 @@ static int mmc_do_hw_reset(struct mmc_host *host, int check)
        }

        /* Set initial state and call mmc_set_ios */
-       mmc_set_initial_state(host);
+       /* TODO: need verify this for UHS2. */
+       if (!host->flags & MMC_UHS2_SUPPORT)
+               mmc_set_initial_state(host);

        mmc_host_clk_release(host);
```

patch原文件drivers/mmc/core/core.c搜索函数名对应的hunk内容，得知代码行数是2287：

```
@@ -2287,7 +2339,9 @@ static int mmc_do_hw_reset(struct mmc_host *host, int check)
 	}
 
 	/* Set initial state and call mmc_set_ios */
-	mmc_set_initial_state(host);
+	/* TODO: need verify this for UHS2. */
+	if (!host->flags & MMC_UHS2_SUPPORT)
+		mmc_set_initial_state(host);
 
 	mmc_host_clk_release(host);
```

去bootlin.com查找[kernel 3.18的core.c代码](https://elixir.bootlin.com/linux/v3.18/source/drivers/mmc/core/core.c)如下(直接搜索drivers/mmc/core/core.c定位到文件，然后在core.c文件ctrl+F查找行数2287)，2287行对不上当然patch fail。

![image-20230313192134365](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202303131921412.png)

根据patch提交时间，其大概率是使用3.18~4.0之间的kernel版本，因此搜寻3.18以后，且符合上面fail点的代码，首先就是[3.19版本](https://elixir.bootlin.com/linux/v3.19/source/drivers/mmc/core/core.c)对比代码如下，可见2287开始的几行和patch完全对应：

![image-20230313191733743](https://raw.githubusercontent.com/cursorhu/blog-images-on-picgo/master/images/202303131917802.png)

打patch也全部通过未报错，所以3.19是此patch可适配的kernel版本：

```
ubuntu@ubuntu-Z390-GAMING-X:~/linux-3.19$ patch -p1 < RFC-1-2-mmc-core-support-UHS-II-in-core-stack..diff
patching file drivers/mmc/core/Makefile
patching file drivers/mmc/core/bus.c
patching file drivers/mmc/core/core.c
patching file drivers/mmc/core/sd.c
patching file drivers/mmc/core/sd_ops.c
patching file drivers/mmc/core/uhs2.c
patching file drivers/mmc/core/uhs2.h
patching file include/linux/mmc/core.h
patching file include/linux/mmc/host.h
patching file include/linux/mmc/uhs2.h
```

 
