# Pydataverse Solution

> Dataverse / pyDataverse 工程问题的持续解决记录。每完成一个问题，经确认后在本文末尾追加完整过程，不仅记录代码答案，也解释问题给客户带来的实际困难和修复后的体验。

## 更新约定

每篇记录必须说明：技术问题、客户现象、受影响角色、复现、客户困难与风险、调查操作、遇到的障碍、沟通方式、最终 solution、测试证据、修复后的客户体验，以及 implemented/tested/approved/merged/released 的准确状态。

不得提交密码、API token、session cookie、私人客户数据或其他敏感信息；客户影响必须基于证据，不得夸大或虚构。

## 目录

1. [Solution 001 — Dataverse Guestbook Cancel 修复与 Terms of Access 回归调查](#solution-001--dataverse-guestbook-cancel-修复与-terms-of-access-回归调查)
2. [Solution 002 — Dataverse.jl CI 去除 Harvard 生产依赖与可配置 `base_url`](#solution-002--dataversejl-ci-去除-harvard-生产依赖与可配置-base_url)

---

## Solution 001 — Dataverse Guestbook Cancel 修复与 Terms of Access 回归调查

> 关联 Issue：[IQSS/dataverse#12205](https://github.com/IQSS/dataverse/issues/12205)  
> 主 PR：[IQSS/dataverse#12220](https://github.com/IQSS/dataverse/pull/12220)  
> 补充 PR：[IQSS/dataverse#12608](https://github.com/IQSS/dataverse/pull/12608)  
> 状态更新时间：2026-08-13

**一句话结论：** Cancel bug 和评审中发现的 Terms of Access 回归都已在 #12220 当前分支中修复并通过 CI；但 #12220 仍未合并进 `develop`，也未进入正式 release，因此代码层面已解决，客户交付层面尚未完成。

### 1. 问题是什么

本次工作包含两个相关但必须分开判断的问题。

#### 1.1 原始问题：Guestbook Response 的 Cancel 按钮无效

Dataverse 数据集可以要求用户在预览、下载或请求访问文件前填写 Guestbook。Issue #12205 报告，在 Dataverse 6.9 的 JSF UI 中，Guestbook Response / Dataset Terms 界面的 **Cancel** 按钮没有按预期退出。

客户看到的现象是：

- 点击 Cancel 后弹窗仍然存在，像是按钮没有反应；或
- 在 Preview Guestbook tab 中无法正常返回 Files tab；
- 客户只能尝试浏览器 Back 等绕过方式。

受影响角色包括所有需要经过 Guestbook 流程的文件使用者，不只限于管理员或数据集所有者。数据集所有者和 repository support 也会间接受影响，因为用户可能报告无法取消下载、预览或访问请求。

#### 1.2 评审中发现的问题：Terms of Access 可能完全不显示

在审查 #12220 相对于 `develop` 的完整 diff 时，发现 Terms of Access 区块有两处与 Cancel 无关的语义变化：

1. 显示条件错误地检查 `disclaimer`，而不是实际显示的 `termsOfAccess`；
2. 标签引用了 `Bundle.properties` 中不存在的 bundle key。

这意味着：数据集明明设置了 Terms of Access，但只要 Disclaimer 为空，下载/请求访问界面就可能完全不显示访问条款。

受影响角色包括：

- 下载或请求受限文件的最终用户；
- 依赖 Terms of Access 告知使用条件的数据集所有者；
- 负责数据访问政策、支持和合规流程的 repository 管理人员。

### 2. 如何复现

#### 2.1 复现 Cancel 问题

前置条件：

1. 创建 Dataverse 和 Dataset；
2. 上传至少一个文件；
3. 给 Dataset 配置包含 required fields 的 Guestbook；
4. 发布 Dataset。

客户操作：

1. 打开已发布 Dataset；
2. 选择 Preview、Download，或在启用 `dataverse.files.guestbook-at-request=true` 时选择 Request Access；
3. Guestbook Response / Dataset Terms 界面出现；
4. 不填写或不提交内容，点击 Cancel。

| 对比 | 结果 |
|---|---|
| 修复前实际结果 | Cancel 没有正确关闭 dialog/tab，用户被留在 Guestbook 流程中 |
| 预期结果 | Dialog 关闭并停留在 Files 页面；Preview 场景返回 Files tab；不提交 response/request |

#### 2.2 复现 Terms of Access 回归

受控复现数据集配置为：

- Dataset 已发布；
- 有 Guestbook；
- 至少有一个 restricted file；
- Disclaimer 为空；
- Terms of Access 设置为唯一测试值 `QA-TOA-SENTINEL-12220`。

客户操作：

1. 打开 Dataset；
2. 选择包含 restricted file 的下载流程；
3. 如果出现 `Inaccessible Files Selected`，点击 Continue；
4. 查看 Dataset Terms dialog。

| 对比 | 结果 |
|---|---|
| 回归版本实际结果 | 只显示 License/Data Use Agreement；Terms of Access 标签和内容完全缺失 |
| 修复版本预期结果 | 显示 `Terms of Access for Restricted Files` 和配置的具体条款 |

### 3. 对客户造成的困难和风险

#### 3.1 Cancel 无效造成的直接困难

- **工作流程被困住：** 客户明确选择取消，但界面没有响应，无法自然回到 Files 页面。
- **需要不直观的 workaround：** 客户必须使用浏览器 Back、关闭 tab 或重新打开 Dataset。这要求客户自己判断如何恢复，而不是由产品提供正常退出路径。
- **丢失上下文和重复操作：** 使用 Back 或重新打开页面可能让客户重新定位文件、重新选择下载方式，增加操作时间。
- **产生不确定感：** 客户无法确认 Cancel 是否已经被系统接收，也不确定 Guestbook response 或 Request Access 是否被提交。
- **降低界面可信度和可访问性：** 一个视觉上可点击但无效果的按钮会让用户认为系统失效；依赖明确导航行为的客户受到的影响更明显。
- **增加支持成本：** 用户可能向 dataset owner 或 repository help desk 报告“无法取消”“页面卡住”。

现有 workaround 是浏览器 Back。它只能帮助部分场景退出，并不是可靠、可发现或符合产品语义的解决方案。

#### 3.2 Terms of Access 缺失造成的客户风险

- **无法做知情决定：** 客户在接受条款、下载或请求访问前看不到数据集所有者配置的访问条件。
- **条款沟通失效：** 数据集所有者虽然完成了 Terms of Access 配置，但 UI 没有传达给最终用户。
- **受限数据流程风险：** 对 restricted files，Terms of Access 通常比普通 License 更具体。缺失会削弱访问条件的可见性和审计解释能力。
- **潜在政策/合规风险：** 如果机构流程要求用户在访问前看到特定条件，隐藏条款可能使流程不符合机构预期。这里记录的是风险，不代表已经确认发生法律或数据泄漏事件。
- **错误标签风险：** 不存在的 bundle key 还可能在某些渲染条件下显示 `???key???` 或不正确标签，进一步让客户困惑。

影响范围不是所有 Dataset，而是满足相关条件的 Dataset：配置了 Terms of Access，并涉及 restricted/request-access 文件；其中 Disclaimer 为空的情况会稳定触发缺失。

### 4. 调查和操作过程

#### 4.1 审查完整 diff

首先把 #12220 与 `develop` 的完整差异逐段比较，而不是只看 Cancel 按钮附近。

发现：

- 文件前半部分有大量缩进/格式变化，增加了审查噪声；
- 真正的 Cancel 修改集中在文件后半部分；
- Terms of Access 的显示条件和 bundle key 也被改变，但与 Cancel 目标无关。

#### 4.2 提交 inline review

在 #12220 中指出：

- `rendered` 应检查 `termsOfAccess`；
- 应保留 restricted/request-access guard；
- 当前 bundle key 不存在；
- 给出可直接应用的两行 suggestion；
- 同时说明 Cancel 按钮本身的 validation gating 逻辑看起来正确。

建议进入共同署名 commit：

```text
e2fe762e — Update src/main/webapp/guestbook-terms-popup-fragment.xhtml
```

#### 4.3 CI 失败与首次回退

`e2fe762e` 的 JSF workflow 中，`13-dataset-actions` 出现 Select All 失败。由于失败发生在修复提交之后，表面上容易把失败归因于这两行。

随后：

```text
257febcb — undo last commit
```

撤销了该建议。短期内检查变绿，但 Terms of Access 回归也被重新带回。

#### 4.4 构造受控 A/B 实验

为了区分“时间上同时出现”和“代码存在因果关系”，建立两个应用镜像：

- **Variant A：** 包含 Terms of Access 两行修复；
- **Variant B：** 与 A 相同，只把 `guestbook-terms-popup-fragment.xhtml` 替换为撤销版本。

固定 Dataverse 版本、PostgreSQL、Solr、测试 runner revision、Node/npm/Playwright 版本、账号、数据和测试命令。A/B 唯一有意差异是目标 XHTML。

#### 4.5 重复运行失败测试

每个 variant：

- 在全新 DB/Solr 环境独立运行 `13-dataset-actions` 三次；
- 再按照 CI 顺序运行 tests 02–13 的前缀一次。

A 和 B 的目标测试都达到 4/4 通过。因此没有证据支持“两行 Terms of Access 修复导致 Select All 失败”。

失败 trace 也没有：

- `filesTable` 的 `toggleSelect` request；
- JSF、EL 或 bundle error；
- Terms of Access fragment 的实际渲染。

该 fixture 没有 Terms of Access、Disclaimer、Guestbook 或 restricted file，因此不会经过目标区块。

#### 4.6 对 Terms of Access 做可逆验证

使用相同数据库、相同数据集和相同浏览器脚本，仅切换应用容器：

```text
B → A → B → A
```

结果始终是：

```text
缺失 → 显示 → 缺失 → 显示
```

这证明错误条件与客户看不到 Terms of Access 之间存在稳定、可逆的因果关系。

#### 4.7 运行完整测试并建立 #12608

在 Variant A 上运行完整 JSF suite，43/43 通过，覆盖 Chromium、Firefox 和 WebKit。

随后创建补充 PR #12608，把两行修复单独展示，并附上：

- 受控 A/B 结论；
- 官方和独立 workflow；
- 43/43 通过结果；
- `xmllint` 和 `git diff --check` 结果。

#### 4.8 提供 Before/After 截图

Philip 表示文字说明不足以快速理解 UI 问题，因此使用同一数据集、同一下载流程生成对照图：

![Terms of Access before and after](assets/solution-001-terms-of-access-before-after.png)

截图让 reviewer 可以直接看到：回归版本没有 Terms of Access，修复版本恢复标签和内容。

#### 4.9 修复回到主 PR

Philip 请 Steven 判断是否把 #12608 加入 #12220。Steven 将等价的两行修复直接应用到主 PR：

```text
bf019c4 — Update src/main/webapp/guestbook-terms-popup-fragment.xhtml
```

之后将最新 `develop` 合并到分支：

```text
ad315c1 — Merge branch 'develop' into 12205-guestbook-response-jsf-popup-cancel-button
```

因为修复已经存在于 #12220，#12608 被关闭为 superseded，避免重复合并。

### 5. 遇到的问题

#### 5.1 Reformat 掩盖语义变化

大量缩进变化让两行逻辑错误很难被发现。解决方法是与 `develop` 做完整语义 diff，并逐项确认与 PR 目标无关的变化。

#### 5.2 单次 CI 红叉造成错误归因

修复提交后恰好发生 Select All 失败，但“修改后失败”不等于“修改导致失败”。解决方法不是继续猜测，而是建立最小差异 A/B 并多次重复。

#### 5.3 为了变绿而回退，重新引入真实客户问题

`257febcb` 消除了当时的可疑提交，却让客户再次看不到 Terms of Access。解决方法是分别验证两个 claim，而不是用一次 CI 结果同时判断所有问题。

#### 5.4 缺少直观 UI 证据

技术说明和测试链接没有让 reviewer 立即理解客户看到什么。解决方法是提供同条件 Before/After 截图。

#### 5.5 补充 PR 与主 PR 重复

#12608 用于隔离和解释两行修复，但最终应由 #12220 统一交付。修复直接应用到 #12220 后及时关闭 #12608。

#### 5.6 历史 commit 红叉仍显示

GitHub Commits 页面保留每个 commit 当时的 checks：

- `e2fe762` 当时有一次 JSF failure；
- `bf019c4` 后很快出现新 commit，旧 JSF/Integration runs 被取消，旧 Read the Docs build 失败；
- 最新 `ad315c1` 的当前检查全部通过。

后续 commit 不会把历史记录改绿。历史红叉是审计记录，不代表当前 HEAD 仍有待解决问题。

### 6. 如何沟通

#### 6.1 使用具体代码和客户结果

没有只说“这两行不对”，而是明确说明：

- 错误字段是 `disclaimer`；
- 应检查 `termsOfAccess`；
- 错误 key 在 bundle 中不存在；
- 客户结果是 Terms of Access 完全缺失；
- 附上可直接应用的 suggestion。

#### 6.2 把三个问题分开

沟通中明确区分：

1. Cancel button bug；
2. Terms of Access regression；
3. `13-dataset-actions` Select All failure。

这样避免因为一个 CI failure 否定另一个已经被证实的 UI 修复。

#### 6.3 用证据替代争论

向维护者提供：

- A/B 测试矩阵；
- trace 中不存在目标 request/error 的事实；
- fixture 不渲染目标区块的说明；
- 43/43 完整测试；
- Before/After screenshot；
- 公开 workflow 链接。

#### 6.4 根据 reviewer 反馈改变表达方式

Philip 表示不理解时，没有继续堆叠长文字，而是补充视觉证据。截图帮助他快速理解“客户原来看不到什么、修复后看到什么”。

#### 6.5 准确描述 GitHub 状态

修复进入 #12220 后使用：

```text
Steven applied the equivalent fix directly to PR #12220,
so PR #12608 was closed as superseded.
```

没有把“修复已应用”误写成“#12220 已合并”。

### 7. 最终 solution

最终 solution 包含两部分。

#### 7.1 Cancel 按上下文执行正确退出行为

Preview tab：

```xhtml
<p:commandButton styleClass="btn btn-default" value="#{bundle.cancel}"
                 rendered="#{popupContext == 'previewTab'}"
                 onclick="history.back();"
                 update="fileForm:tabView">
</p:commandButton>
```

普通 Download dialog：

```xhtml
<p:commandButton styleClass="btn btn-default" value="#{bundle.cancel}"
                 rendered="#{popupContext != 'previewTab'}"
                 onclick="PF('guestbookAndTermsPopup').hide();">
</p:commandButton>
```

Request Access 的 Cancel 同样关闭 `guestbookAndTermsPopup`，不提交访问请求。

只有 Accept / Request Access 等真正提交操作的按钮传入：

```xhtml
<f:param name="DO_GB_VALIDATION_#{popupContext}" value="true"/>
```

Cancel 不传该参数，因此不会被 Guestbook required fields 阻止，也不会因为取消动作触发提交校验。

#### 7.2 恢复正确 Terms of Access 渲染

```xhtml
<div class="form-group"
     jsf:rendered="#{!empty workingVersion.termsOfUseAndAccess.termsOfAccess and (hasRestrictedFile or !empty fileDownloadHelper.filesForRequestAccess)}">
    <label jsf:for="fdTermsOfAccess" class="col-sm-3 control-label">
        #{bundle['file.dataFilesTab.terms.list.termsOfAccess.termsOfsAccess']}
    </label>
```

这确保：

- 根据真正显示的 `termsOfAccess` 决定是否渲染；
- 只在 restricted/request-access 相关场景显示；
- 使用项目中实际存在的 bundle key。

### 8. 测试和验证

#### 8.1 Select All 失败的 A/B 排查

| Variant | 全新环境独立运行 | CI 顺序前缀运行 | `13-dataset-actions` 合计 |
|---|---:|---:|---:|
| A：包含两行修复 | 3/3 | 1/1 | 4/4 通过 |
| B：撤销两行修复 | 3/3 | 1/1 | 4/4 通过 |

结论：两行修复没有对该失败表现出可观察影响。

#### 8.2 Terms of Access 直接 A/B

| 顺序 | Variant | 客户在 dialog 中看到的结果 |
|---:|---|---|
| 1 | B（回归版） | ❌ 条款缺失 |
| 2 | A（修复版） | ✅ 标签和 sentinel 显示 |
| 3 | B（回归版） | ❌ 条款再次缺失 |
| 4 | A（修复版） | ✅ 条款再次恢复 |

结论：回归和修复均可稳定复现。

#### 8.3 完整和官方 CI

- JSF：**43/43 passed**，覆盖 Chromium、Firefox、WebKit，并包括 `13-dataset-actions`；
- Container Integration Tests：passed；
- CodeQL：passed；
- Read the Docs：passed；
- Test Results：404 tests，389 passed，15 skipped，0 failed。

公共证据：

- 最新 JSF：<https://github.com/IQSS/dataverse/actions/runs/31705546363>
- 最新 Integration：<https://github.com/IQSS/dataverse/actions/runs/31705546389>
- 曾失败的 JSF：<https://github.com/IQSS/dataverse/actions/runs/31608775356>
- 主 PR：<https://github.com/IQSS/dataverse/pull/12220>

#### 8.4 验证边界

这次调查最严格的直接 A/B 证据针对 Terms of Access 回归及其与 Select All failure 的关系。主 PR 的 Cancel 实现和完整回归测试已通过，但 upstream suite 中没有为 Preview、Download、Request Access 三个 Cancel 路径分别新增独立的专用 E2E assertion。未来可以补充这三个明确的 UI tests，减少同类回归风险。

原始 A/B 产物约 1.7GB，保留在本地实验环境，没有上传到公开仓库；本文保留了方法、关键结果、截图和公共 CI 链接。

### 9. 修复后的客户体验

当 #12220 最终合并并随版本发布后，预期客户体验为：

- **Download：** 点击 Cancel 后弹窗关闭，客户留在 Dataset Files 页面；
- **Preview：** 点击 Cancel 后离开 Guestbook Response，返回 Files tab；
- **Request Access：** 点击 Cancel 后弹窗关闭，不发出访问请求；
- **表单校验：** 客户选择取消时，不会被 Name、Email 等 required fields 阻止；
- **条款可见性：** 涉及 restricted/request-access 文件时，客户能够看到数据集所有者配置的 Terms of Access；
- **界面可信度：** 按钮行为符合文字含义，客户不必猜测是否已取消；
- **支持成本：** 减少“Cancel 没反应”“看不到访问条款”等支持请求；
- **政策沟通：** 数据集所有者配置的访问条件能够在正确流程中传达给客户。

需要强调：这些改善目前存在于 PR 分支和预览/测试环境。因为尚未 merge/release，使用正式已发布 Dataverse 版本的客户还不能假定已经获得该修复。

### 10. 是否已合并和发布

截至 2026-08-13 的准确状态：

| 阶段 | 状态 | 说明 |
|---|---|---|
| 已实现 | ✅ 是 | Cancel 和 Terms of Access 修复均在 #12220 HEAD `ad315c1f` |
| 已测试 | ✅ 是 | A/B、完整 JSF 和 Integration 已完成 |
| CI 通过 | ✅ 是 | 当前 HEAD 所有检查通过 |
| Review approved | ✅ 是 | PR 状态为 Approved |
| Mergeable | ✅ 是 | GitHub 显示 Mergeable / Clean |
| 已合并进 `develop` | ❌ 否 | #12220 仍为 Open |
| Issue 已关闭 | ❌ 否 | #12205 仍为 Open |
| 已进入正式 release | ❌ 否 | 最新正式 release 为 v6.11，未包含这个 Open PR |
| 补充 PR #12608 | ✅ 已关闭 | 未单独 merge；因等价修复已进入 #12220 而 superseded |

最终判断：

> **The fix is implemented, validated, approved, and ready to merge. It is not yet merged or released to customers.**

### 11. 可复用经验

1. **先写客户现象，再写代码。** Reviewer 和 support 需要先知道客户无法完成什么，而不仅是哪个表达式写错。
2. **大型 reformat 中逐项检查语义。** 格式变化会掩盖两行高影响逻辑错误。
3. **时间相关不等于因果关系。** 修改后 CI 失败时，用最小差异 A/B 验证，不凭一次红叉决定回退。
4. **回退也需要回归分析。** 让 CI 变绿的 revert 可能重新引入客户可见 bug。
5. **把独立问题分开表达。** Cancel、Terms of Access、Select All failure 需要各自的证据和结论。
6. **UI 问题优先提供 Before/After。** 相同数据和流程的截图能快速解释客户影响。
7. **状态用词必须精确。** Applied、cherry-picked、superseded、merged、released 含义不同。
8. **当前 HEAD 全绿即可。** 历史 commit 红叉是不可改写的审计记录，不代表当前 PR 未解决。
9. **明确验证边界。** 完整 suite 通过不等于每条客户路径都有专用断言；应诚实记录尚可增强的测试。
10. **文档要区分技术完成与客户交付。** 只有 merge 并 release 后，正式版本客户才真正获得修复。

## Solution 002 — Dataverse.jl CI 去除 Harvard 生产依赖与可配置 `base_url`

> 关联 Issue：[gdcc/Dataverse.jl#36](https://github.com/gdcc/Dataverse.jl/issues/36)<br>
> 关联 PR：[gdcc/Dataverse.jl#38](https://github.com/gdcc/Dataverse.jl/pull/38)<br>
> 合并 commit：[17b4d9e](https://github.com/gdcc/Dataverse.jl/commit/17b4d9ecc8babe26fdd18dfe636baf6372af2104)<br>
> 最终 upstream CI：[run 31813363863](https://github.com/gdcc/Dataverse.jl/actions/runs/31813363863)<br>
> 状态更新时间：2026-08-14

**一句话结论：** Dataverse.jl 的阻塞 CI 已不再读取 Harvard Dataverse 生产数据，而是针对每个 job 启动隔离 Dataverse 并动态创建、发布和下载 fixture；同时保留旧调用默认行为，允许通过 `base_url` 连接其他实例，并把空响应、HTTP 错误和无效 JSON 转换为包含上下文的可操作错误。PR #38 已合并并关闭 Issue #36，但截至本文更新时间尚未进入 Dataverse.jl 正式 release。

### 1. 问题是什么

#### 1.1 技术问题

Dataverse.jl 的 REST API、下载 URL、JSON-LD 和测试流程长期假定服务器是：

```text
https://dataverse.harvard.edu
```

CI 还依赖 Harvard 上的固定 collection、DOI 和 file，例如：

```text
ECCOv4r2
doi:10.7910/DVN/RNXA2A
doi:10.7910/DVN/3HPRZI
```

调查时，GitHub CI 对 Harvard Dataverse 的请求收到：

```text
HTTP 202
Content-Type: text/html
x-amzn-waf-action: challenge
response body: empty
```

原代码立即执行 `JSON.parse`，因此最终只显示：

```text
ArgumentError: invalid JSON ... UnexpectedEOF
```

这个异常没有告诉使用者真实 URL、HTTP 状态、Content-Type 或空 body，容易被误判为 Dataverse 返回了损坏 JSON 或 Julia JSON parser 出错。

#### 1.2 客户和贡献者看到的现象

这里的主要“客户”是 Dataverse.jl 的使用者、贡献者、维护者及依赖它的 Julia package 开发者。

他们看到的现象包括：

- 与本次代码修改无关的 PR 因生产 Dataverse 的 WAF/可用性而失败；
- 同一失败影响 Dataverse.jl 以及引用特定海洋数据集的 Climatology.jl、Drifters.jl、MITgcm.jl 等工作；
- 错误只显示 `UnexpectedEOF`，无法直接判断是 WAF challenge、空 body 还是 API 错误；
- 机构 Dataverse、测试 Dataverse 和本地实例使用者无法在所有 Julia API 路径中一致指定自己的服务器；
- 文档构建执行真实网络示例，生产服务器波动也可能阻断文档发布。

本次事件没有证据表明发生了数据损坏、权限泄漏或隐私事件。影响集中在 CI/release 可靠性、开发效率、错误诊断和替代实例可用性。

### 2. 如何复现

#### 2.1 原始失败

在能够触发 Harvard WAF challenge 的 CI 环境中调用：

```julia
using Dataverse
Dataverse.dataverse_scan(:ECCOv4r2)
```

或：

```julia
Dataverse.file_list("doi:10.7910/DVN/RNXA2A")
```

原始流程为：

1. 构造 Harvard API URL；
2. `HTTP.get` 得到 HTTP 202、`text/html`、空 body；
3. 不检查响应状态和 body，直接 `JSON.parse`；
4. 抛出 `UnexpectedEOF`。

| 对比 | 结果 |
|---|---|
| 修复前实际结果 | CI 依赖生产 Harvard；challenge/空响应被报告为无上下文的 JSON EOF |
| 预期结果 | 阻塞 CI 使用隔离 fixture；API 异常包含 URL、status、Content-Type 和 body 情况 |

#### 2.2 为什么不能只替换为 demo.dataverse.org

直接把 hostname 改为 `https://demo.dataverse.org` 仍会失败，因为当前测试使用的 `ECCOv4r2` collection 和固定 DOI 不存在于 demo。

仅替换 hostname 不能解决以下耦合：

- fixture identity；
- dataset 发布状态；
- file ID；
- 下载 URL；
- 测试数据生命周期。

因此 fixture 必须在测试中动态创建，而不能继续假设某个公共服务器永久保存固定数据。

### 3. 对客户造成的困难和风险

#### 3.1 工作流程和维护成本

- **PR 被非代码因素阻断：** 贡献者即使没有修改 Dataverse API，也必须等待 Harvard 恢复或反复重跑。
- **错误诊断成本高：** `UnexpectedEOF` 隐藏了 HTTP 202 和 WAF challenge，维护者需要额外抓取 header/body 才能找到根因。
- **下游交付延迟：** 依赖 package 的测试、合并和 release 可能被同一个生产端点间接拖慢。
- **替代实例成本高：** 使用本地或机构 Dataverse 的客户需要 patch 源码或手工重建下载 URL，容易遗漏内部调用路径。
- **文档不稳定：** 构建文档时执行生产请求，会把外部服务状态变成文档发布的前置条件。

#### 3.2 原有 workaround 及代价

可用但不可靠的临时 workaround 包括：

1. 重跑 CI：只有 challenge 短暂消失时才有效，无法提供确定性；
2. 等待 Harvard 恢复：开发者没有恢复时间控制权；
3. 把固定文件复制到 demo：需要账号/token、同步任务、数据一致性检查和长期维护；
4. 在源码中替换 hostname：demo 没有原 fixture，且容易漏掉下载和 JSON-LD 路径；
5. 禁用失败测试：会失去真实 Dataverse API 兼容性覆盖。

对仅使用 Harvard 默认 API 的现有用户，本 PR 不改变默认服务器；Harvard 自身不可用时，他们仍需要等待服务恢复或显式选择其他 `base_url`。本 PR 解决的是可配置性、诊断质量和项目自身 CI 的生产耦合，不承诺消除所有外部数据依赖。

### 4. 调查和操作过程

#### 4.1 建立基线并定位真实响应

首先运行现有测试并检查失败栈，然后使用直接 HTTP 请求确认：

- status 是 202，不是正常 Dataverse JSON response；
- Content-Type 是 `text/html`；
- body 为空；
- header 明确显示 WAF challenge。

这把问题从“JSON parser 可能有 bug”缩小为“客户端没有验证 HTTP 响应，并依赖受 WAF 影响的生产服务”。

#### 4.2 清点所有硬编码路径

检查了：

- `src/restDataverse.jl`；
- `src/downloads.jl`；
- `src/json_ld.jl`；
- PythonCall extension；
- `test/runtests.jl`；
- Documenter 首页和 Pluto notebook；
- CI workflow。

关键判断是：只给顶层 `file_list` 换 URL 不够。`files_to_DataFrame` 会生成下载 URL，`file_download` 会再次调用列表 API，JSON-LD 也有独立页面 URL；这些路径必须共享同一 `base_url`。

#### 4.3 在 Issue 中提出范围

向维护者解释：

- Harvard 返回 202 WAF challenge 和空 HTML；
- demo 不包含现有 fixtures；
- 建议增加 `base_url`、隔离测试、响应校验和离线文档；
- 保持默认 Harvard 行为以避免破坏兼容性。

Gael 接受了该范围。Philip 建议使用 [`gdcc/dataverse-action`](https://github.com/gdcc/dataverse-action)，并给出 pyDataverse workflow 作为参考，因此没有另造不完整 mock server。

#### 4.4 选择混合测试策略

最终设计分为三层：

1. **离线 unit tests：** 直接构造 HTTP responses，验证空 body、404、无效 JSON 和 URL 生成；
2. **真实 integration test：** 使用 `dataverse-action` 启动隔离 Dataverse；
3. **离线 documentation：** 示例不在构建时访问生产，Pluto export 使用确定性 sample DataFrame。

这样既避免生产耦合，也没有用 mock 替代所有真实 API 行为。

#### 4.5 分阶段实现

主要 commits：

```text
f20bab4 — Add configurable Dataverse base URLs
c3923ff — Test against an isolated Dataverse instance
f108ca2 — Avoid extension method overwrites
dd78d7c — Match integration fixture DOI URLs
b29362e — Keep local documentation builds offline
```

实现内容包括：

- URL normalization/build helpers；
- 集中的 JSON response validation；
- REST、download、JSON-LD 的 `base_url` 传播；
- dynamic fixture module；
- CI action outputs 到测试环境变量；
- 离线文档和 notebook fixture；
- PythonCall extension generic-function 声明，避免 precompile method overwrite。

#### 4.6 动态创建 integration fixture

每次 integration run 都会：

1. 生成唯一 collection alias；
2. 创建并发布 collection；
3. 创建最小 dataset metadata；
4. 上传小型文本文件；
5. 发布 dataset；
6. poll 到 `RELEASED`；
7. 从响应读取 persistent ID 和 file ID；
8. 调用 Dataverse.jl 扫描、列举和下载；
9. 比较下载后的 bytes。

没有把临时 DOI、file ID 或 API token 写入仓库。

#### 4.7 首轮 GitHub CI 失败的定位

首轮 fork CI 中：

- Documentation 通过；
- Julia 1.9 和最新版均成功启动隔离 Dataverse；
- integration test 10/11 通过；
- 唯一失败是字符串断言：

```text
fixture:      doi:10.5072/FK2/...
API response: https://doi.org/10.5072/FK2/...
```

这证明 `dataverse-action`、bootstrap、publish、list 和 download 都正常；失败来自测试对同一 DOI 的两种标准表示处理不一致，而不是基础设施问题。

修复为比较规范化后的 persistent URL 后，第二轮和最终轮均全绿。

#### 4.8 本地环境与 upstream workflow

本地隔离 Dataverse 测试和文档构建通过。调查期间还遇到 PythonCall precompile 耗时和磁盘空间不足，这些属于本机资源问题，没有错误归因给 `dataverse-action` 或请求 Philip 支援。

正式 upstream PR 首次运行需要维护者批准 fork workflow。批准后所有 jobs 通过；Gael 回复 `LGTM -- thank you very much!`，并于 2026-08-14 合并 PR #38，Issue #36 随即关闭。

### 5. 遇到的问题

#### 5.1 生产服务返回 2xx 但不是可用 API response

HTTP 202 属于 2xx，单纯依赖 `status_exception` 不足以发现问题。客户端还必须验证 body 非空并能解析为 JSON，同时在失败信息中保留响应上下文。

#### 5.2 demo 缺少原始 fixtures

公共 demo 不是 Harvard 数据集的镜像。解决方法是让 CI 自己拥有 fixture 生命周期，而不是创建月度复制任务。

#### 5.3 发布后的 DOI 表示不同

创建 API 返回 `doi:` persistent ID，而 collection contents 返回 `https://doi.org/` URL。测试最初把两种合法表示当成字面相同字符串。解决方法是规范化预期 URL，再进行精确比较。

#### 5.4 测试需要写权限，但 package API 主要是匿名读取

integration bootstrap 需要 API token 创建和发布 fixture，但产品 read/list/download 路径不应为了测试而扩大成完整 authenticated client。token 只存在于 CI environment，并由 fixture helper 使用；被测试的 package API 仍执行已发布数据的匿名读取。

#### 5.5 Python extension precompile overwrite

原 package 同时定义 fallback 方法和 extension default method，触发 overwrite/precompile 问题。改为在主 module 声明空 generic function，由 extension 提供实现，保持调用兼容并消除 overwrite。

#### 5.6 文档“离线”不能只依赖 GitHub 的 `CI=true`

如果 notebook 只在 `CI=true` 时使用 sample data，本地 Documenter build 仍可能访问 Harvard。最终由 `docs/make.jl` 显式设置 `DATAVERSE_OFFLINE_DOCS=true`，确保本地和 GitHub 文档导出使用同一离线行为。

#### 5.7 首次贡献者 workflow 需要批准

upstream run 最初显示 `action_required`，这是 GitHub 对 fork PR 的标准权限流程，不是 action 启动故障。维护者批准后原 commit 直接通过，无需代码 workaround。

### 6. 如何沟通

#### 6.1 先提供协议层证据

没有只报告“Harvard 挂了”，而是提供 status、Content-Type、空 body 和 WAF header，让维护者能够区分服务器内容、客户端解析和 CI 网络问题。

#### 6.2 明确说明 demo 为什么不够

回复 Philip 时说明原 collection 和 DOI 不在 demo，避免把问题简化为 hostname replacement。随后接受他提出的 `dataverse-action`，而不是坚持原先的 mock server 表述。

#### 6.3 在请求支援前先判断责任边界

两次测试 job 都成功启动 action，失败点是 DOI assertion，因此没有把本地资源问题或测试代码问题升级给 Philip。只有 action output、bootstrap 或 Dataverse API 本身出现可复现异常时才需要维护者支援。

#### 6.4 PR 中解释客户影响、兼容性和剩余范围

PR body 明确说明：

- 为什么生产依赖会阻断贡献者；
- 默认 Harvard 行为保持不变；
- Docker startup 增加时间，但换来确定 fixture；
- Zenodo/Natural Earth archive tests 未在本 PR 修改；
- 下游海洋数据集迁移是独立问题。

#### 6.5 用预验证降低 reviewer 风险

正式 upstream PR 前，先在 fork Draft PR 验证 action、Julia 1.9、最新 Julia 和文档。upstream workflow 等待批准时，提供了已通过的 fork run 链接，维护者不需要盲目批准未经验证的 Docker workflow。

### 7. 最终 solution

#### 7.1 集中构造 Dataverse URL

```julia
const DEFAULT_BASE_URL = "https://dataverse.harvard.edu"

function _normalize_base_url(base_url::AbstractString)
    normalized = rstrip(strip(String(base_url)), '/')
    isempty(normalized) && throw(ArgumentError("Dataverse base_url must not be empty."))
    normalized
end

function _build_url(base_url::AbstractString, path::AbstractString; query=Dict{String,String}())
    url = string(_normalize_base_url(base_url), "/", lstrip(String(path), '/'))
    string(HTTP.URI(url; query=query))
end
```

这保证 trailing slash、endpoint path 和 query encoding 由同一逻辑处理。

#### 7.2 在 JSON parse 前验证 response

```julia
function _get_json(path::AbstractString; base_url=DEFAULT_BASE_URL, query=Dict{String,String}())
    url = _build_url(base_url, path; query=query)
    response = HTTP.get(url; status_exception=false)
    _parse_json_response(response, url)
end
```

`_parse_json_response` 分别处理：

- 非 2xx status；
- 空或全空白 body；
- 无效 JSON；
- 正常 JSON，即使 proxy 提供不理想的 Content-Type 也保持兼容。

错误包含 URL、HTTP status 和 Content-Type；空 body 会明确写出 `empty response body; expected JSON`，不再暴露原始 `UnexpectedEOF`。

#### 7.3 贯穿公共 API 的 `base_url`

兼容调用仍然有效：

```julia
Dataverse.file_list("doi:10.7910/DVN/EE3C40")
```

其他实例可以使用：

```julia
files = Dataverse.file_list(
    "doi:10.1234/EXAMPLE";
    base_url="https://data.example.edu",
)

header, dataverses, datasets = Dataverse.dataverse_scan(
    :root;
    base_url="https://data.example.edu",
)

Dataverse.file_download(
    "doi:10.1234/EXAMPLE",
    "example.txt";
    base_url="https://data.example.edu",
)
```

`files_to_DataFrame` 也使用选定实例构造 `/api/access/datafile/<id>`，避免 list 请求走本地实例、download URL 却静默回到 Harvard。

#### 7.4 隔离 Dataverse CI

workflow 使用：

```yaml
- name: Start isolated Dataverse
  id: dataverse
  uses: gdcc/dataverse-action@main

- uses: julia-actions/julia-runtest@v1
  env:
    DATAVERSE_BASE_URL: ${{ steps.dataverse.outputs.base_url }}
    DATAVERSE_API_TOKEN: ${{ steps.dataverse.outputs.api_token }}
```

测试只从 action outputs 获取临时 URL/token，所有 fixture identifiers 从 API response 动态读取。

#### 7.5 文档构建不访问生产 Dataverse

首页网络示例改为非执行 `julia` code block。Pluto notebook 在导出时使用固定 DataFrame；`docs/make.jl` 显式启用离线模式。因此文档仍展示真实 API 用法，但 build 本身不要求 Harvard 在线。

### 8. 测试和验证

#### 8.1 离线 unit tests

覆盖：

- base URL 去空格和 trailing slash；
- 空 `base_url`；
- persistent ID query encoding；
- 正常 JSON；
- HTTP 200、非理想 Content-Type 但 body 是合法 JSON；
- HTTP 202、`text/html`、空 body；
- HTTP 404；
- HTTP 200、无效 JSON；
- download URL 使用所选 Dataverse 实例。

#### 8.2 真实 integration test

验证：

- 创建并发布唯一 collection；
- 创建 dataset 并上传文本文件；
- poll 到 `RELEASED`；
- `dataverse_scan` 找到动态 dataset；
- `file_list` 返回动态 filename/file ID；
- download URL 不包含 Harvard hostname；
- `file_download` 下载成功；
- 下载 bytes 与上传内容完全一致。

#### 8.3 CI 结果

最终 upstream workflow：

| Job | 结果 |
|---|---|
| Julia 1.9 — Ubuntu x64 | ✅ Success |
| 最新 Julia — Ubuntu x64 | ✅ Success |
| Documentation | ✅ Success |

公共证据：

- upstream final run：<https://github.com/gdcc/Dataverse.jl/actions/runs/31813363863>
- merge commit：<https://github.com/gdcc/Dataverse.jl/commit/17b4d9ecc8babe26fdd18dfe636baf6372af2104>
- maintainer acceptance：<https://github.com/gdcc/Dataverse.jl/pull/38#issuecomment-5295980327>

#### 8.4 验证边界和剩余风险

- Codecov 报告 patch coverage 约 73.47%，主要缺口是不会在文档/CI 中访问真实页面的 JSON-LD 路径；required checks 仍全部通过。未来可用本地 HTTP fixture 增加 JSON-LD coverage。
- 现有 archive tests 仍下载 Zenodo 和 Natural Earth；它们不是 Harvard 依赖，但仍是外部网络风险，可在后续 PR 改为本地 synthetic archives。
- Climatology.jl、Drifters.jl、MITgcm.jl 等若有意读取 Harvard 上的特定科学数据，仍需要独立的数据镜像/fixture 方案。
- `dataverse-action@main` 及其默认 `unstable` images 是可变依赖。此次采用项目维护者推荐、pyDataverse 已使用的方式；未来如 action 提供稳定 tag，可再评估 pinning。
- 本 PR 测试匿名读取已发布数据，没有扩展 package 的 authenticated write API。

### 9. 修复后的客户体验

合并后的开发和 API 使用体验为：

- **贡献者：** Dataverse.jl 的核心 API 测试不再因为 Harvard WAF 或生产维护而失败；fixture 每次由 CI 自己创建。
- **维护者：** 失败可以区分 HTTP status、空 body 和无效 JSON，减少日志反查和误判。
- **机构/本地实例使用者：** 同一套 list、scan、download 和 JSON-LD API 可以指向自己的 Dataverse，不必 fork package 修改 hostname。
- **文档维护者：** 首页和 Pluto export 不再要求生产数据可访问。
- **现有 Harvard 用户：** 不提供 `base_url` 时行为保持兼容；不需要迁移现有调用。
- **下游 package：** 现在具备改用隔离或机构 Dataverse 的基础，但依赖特定 Harvard 科学数据的 package 不会被本 PR 自动迁移。

错误体验也发生变化：客户不再只看到 parser 的 `UnexpectedEOF`，而会看到哪个 URL 返回了什么 status/Content-Type，以及 body 是否为空，从而更快决定是重试、联系实例管理员还是修正 endpoint。

由于 PR 已合并但尚未发布，使用 registry 中 v0.2.8 的客户还没有获得这些变化；需要等待包含 merge commit `17b4d9e` 的后续 Dataverse.jl release，或直接使用包含该 commit 的开发版本。

### 10. 是否已合并和发布

截至 2026-08-14 的准确状态：

| 阶段 | 状态 | 说明 |
|---|---|---|
| 已实现 | ✅ 是 | 5 个 commits 完成 URL、response、fixture、CI 和 docs 修改 |
| 已测试 | ✅ 是 | 离线 unit、真实 Dataverse integration、docs 均完成 |
| CI 通过 | ✅ 是 | Julia 1.9、最新 Julia、Documentation 全绿 |
| Maintainer accepted | ✅ 是 | Gael 回复 `LGTM` 并执行 merge |
| 已合并 | ✅ 是 | PR #38 于 2026-08-14 合并为 `17b4d9e` |
| Issue 已关闭 | ✅ 是 | Issue #36 随 PR merge 自动关闭 |
| 已进入正式 release | ❌ 否 | 最新 release 仍为 v0.2.8，对应 merge 前 commit `177b81a` |

最终判断：

> **The fix is implemented, tested, CI-passed, accepted, and merged. It is not yet released in a tagged Dataverse.jl version.**

### 11. 可复用经验

1. **CI 不应把公共生产服务当 fixture。** 即使 API 是只读的，WAF、维护和限流也会使无关 PR 失败。
2. **先看 HTTP 层，再看 parser。** status、Content-Type、headers 和 body condition 能快速区分服务端 challenge 与 JSON bug。
3. **更换 hostname 不等于解除 fixture 耦合。** collection、DOI、file ID 和发布状态都必须纳入测试数据设计。
4. **unit mock 与真实 integration 各有职责。** 错误分支用离线 response tests，API compatibility 用隔离 Dataverse。
5. **动态 fixture 不保存临时 ID。** DOI 和 file ID 应从 API response 获取，避免跨环境和跨运行漂移。
6. **配置必须贯穿内部调用。** 顶层 `base_url` 如果没有传入下载 URL builder，仍会产生隐蔽的生产访问。
7. **保留兼容默认值降低迁移成本。** 新能力通过 keyword 提供，不强迫现有 Harvard 用户立即改代码。
8. **2xx 不代表 body 可用。** 仍需检查空 body 和 JSON parse，并提供响应上下文。
9. **首次 CI 红叉先判断 action 是否真的失败。** 本次 action 已启动且 10/11 integration assertions 通过，根因只是 DOI 表示差异。
10. **本机资源问题不要升级为 upstream 基础设施问题。** 先隔离磁盘、precompile 和 action/API 责任边界。
11. **文档离线模式要显式。** 只依赖 GitHub 的 `CI` 环境变量，会让本地 docs build 继续访问生产。
12. **准确区分 merged 与 released。** 合并解决了仓库代码问题，但 registry 用户要等新 tag 才获得修复。
13. **下游数据可用性要单独治理。** package CI 隔离与长期科学数据托管是两个问题，不应在一个 PR 中混合。
