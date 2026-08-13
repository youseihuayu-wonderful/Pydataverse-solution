# Pydataverse Solution

> Dataverse / pyDataverse 工程问题的持续解决记录。每完成一个问题，经确认后在本文末尾追加完整过程，不仅记录代码答案，也解释问题给客户带来的实际困难和修复后的体验。

## 更新约定

每篇记录必须说明：技术问题、客户现象、受影响角色、复现、客户困难与风险、调查操作、遇到的障碍、沟通方式、最终 solution、测试证据、修复后的客户体验，以及 implemented/tested/approved/merged/released 的准确状态。

不得提交密码、API token、session cookie、私人客户数据或其他敏感信息；客户影响必须基于证据，不得夸大或虚构。

## 目录

1. [Solution 001 — Dataverse Guestbook Cancel 修复与 Terms of Access 回归调查](#solution-001--dataverse-guestbook-cancel-修复与-terms-of-access-回归调查)

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
