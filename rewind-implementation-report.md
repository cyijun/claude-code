# Claude Code `/rewind` 功能实现报告

## 1. 功能定位

`/rewind`（别名 `/checkpoint`）是一个 **local 命令**，注册在 `package/restored-src/src/commands.ts` 中。它的作用是把代码和/或对话回退到某个历史用户消息之前的状态。真正的实现分为两条线：

- **对话回退**：把消息列表截断到指定消息之前，并清理相关状态。
- **代码回退**：把被追踪的文件恢复到指定消息发送时的版本。

命令入口非常简单（`commands/rewind/rewind.ts`），只是唤起 `MessageSelector` UI：

```ts
export async function call(...) {
  if (context.openMessageSelector) {
    context.openMessageSelector()
  }
  return { type: 'skip' }
}
```

## 2. 用户交互流程

用户在 REPL 中输入 `/rewind` 后，会弹出 `MessageSelector`（`components/MessageSelector.tsx`）。它列出所有可选的用户消息，并允许选择回退范围：

- `Restore code and conversation`：同时回退代码和对话。
- `Restore conversation`：只回退对话，保留代码。
- `Restore code`：只回退代码，保留对话。
- `Summarize from here / up to here`：把某段对话压缩成摘要，不是真正的回退。
- `Never mind`：取消。

选择消息后，组件调用两个回调：

- `onRestoreCode(message)` → 调用 `fileHistoryRewind(..., message.uuid)` 恢复文件。
- `onRestoreMessage(message)` → 调用 `rewindConversationTo(message)` 截断对话并恢复输入框内容。

## 3. 代码回退的实现：`fileHistory.ts`

代码回退的核心在 `package/restored-src/src/utils/fileHistory.ts`。它维护了一个文件历史系统。

### 3.1 数据结构

```ts
export type FileHistoryState = {
  snapshots: FileHistorySnapshot[]
  trackedFiles: Set<string>   // 被追踪的相对路径
  snapshotSequence: number
}

export type FileHistorySnapshot = {
  messageId: UUID
  trackedFileBackups: Record<string, FileHistoryBackup>
  timestamp: Date
}

export type FileHistoryBackup = {
  backupFileName: string | null  // null 表示该快照点文件不存在
  version: number
  backupTime: Date
}
```

- `trackedFiles`：所有曾经被编辑过的文件路径集合。
- `snapshots`：每个用户消息对应一个快照，记录当时每个被追踪文件的备份版本。
- 备份文件存放在 `~/.claude/file-history/{sessionId}/{hash}@v{version}`。

### 3.2 文件何时被追踪

文件不是一开始就追踪所有项目文件，而是 **第一次被编辑前** 加入追踪。目前会在修改文件前调用 `fileHistoryTrackEdit` 的工具有：

- `FileEditTool`（行内编辑）
- `FileWriteTool`（整文件写入）
- `NotebookEditTool`（Notebook 单元格编辑）
- `BashTool`（**仅当它识别并模拟的 `sed -i` 原地替换命令**）

也就是说，BashTool 里普通的 `echo >> file`、`python script.py`、手动 `mv/cp/rm` 等命令**不会被追踪**；只有在 Claude 把 `sed -i` 解析为模拟编辑时才会走 `applySedEdit` 并调用 `fileHistoryTrackEdit`。

#### BashTool 对 `sed -i` 的特殊处理

BashTool 并不是把所有 `sed -i` 都交给 shell 执行，而是对**简单且可被解析的** `sed -i 's/pattern/replacement/flags' file` 做了一套“模拟执行 + 权限预览”流程：

1. **权限对话框分流**：`BashPermissionRequest.tsx:89` 用 `parseSedEditCommand(command)` 解析命令。如果能解析成功，就渲染 `SedEditPermissionRequest`，而不是普通的 Bash 执行确认框。
2. **预计算新内容**：`SedEditPermissionRequest.tsx:108` 读取文件原内容，并用 `applySedSubstitution(oldContent, sedInfo)` 在 JavaScript 里算出替换后的内容。
3. **展示 diff**：用 `FileEditToolDiff` 把 old/new 对比展示给用户，就像 `FileEditTool` 一样。
4. **用户确认后注入模拟结果**：如果用户批准，`SedEditPermissionRequest.tsx:160` 会往 `BashTool` 输入里塞一个内部字段 `_simulatedSedEdit: { filePath, newContent }`。
5. **BashTool 直接写入，不跑 shell**：`BashTool.tsx:627` 检测到 `_simulatedSedEdit` 后，直接调用 `applySedEdit(...)`。该函数：
   - 读取原内容（用于 VSCode 通知）；
   - 调用 `fileHistoryTrackEdit(...)` 备份原内容；
   - 用 `writeTextContent` 把预计算好的 `newContent` 写回文件；
   - 通知 VSCode 文件已更新。

如果 `sed` 命令不符合 `parseSedEditCommand` 的受限模式（例如用了多个文件、复杂脚本、`-e` 链、非 `s///` 表达式等），BashTool 会走普通 shell 执行路径，**不会**记录文件历史。

示例（`FileEditTool.ts:435`）：

```ts
if (fileHistoryEnabled()) {
  await fileHistoryTrackEdit(
    updateFileHistoryState,
    absoluteFilePath,
    parentMessage.uuid,
  )
}
```

`fileHistoryTrackEdit` 的工作：

1. 把路径转成相对路径作为 `trackingPath`。
2. 检查当前最新快照是否已记录该文件；如果已记录则跳过，避免重复备份。
3. 为当前文件内容创建 **v1 备份**（如果文件不存在则记录 `null`）。
4. 把该文件加入 `trackedFiles`，并把备份写入当前快照。

### 3.3 快照何时生成

每次用户提交消息后，处理完本地命令/输入后，会调用 `fileHistoryMakeSnapshot`（`utils/handlePromptSubmit.ts:528`）：

```ts
newMessages.filter(selectableUserMessagesFilter).forEach(message => {
  void fileHistoryMakeSnapshot(
    (updater) => setAppState(prev => ({ ...prev, fileHistory: updater(prev.fileHistory) })),
    message.uuid,
  )
})
```

`fileHistoryMakeSnapshot` 的逻辑：

1. 遍历所有 `trackedFiles`。
2. 对每个文件：
   - `stat` 文件；如果文件已删除，则记录 `backupFileName: null`。
   - 如果文件未变（通过 mtime/size/mode 或内容比较），复用上一个快照的备份。
   - 如果文件有变化，创建新版本备份（`version + 1`）。
3. 生成新的 `FileHistorySnapshot`，挂到 `messageId` 上。
4. 快照数量上限为 `MAX_SNAPSHOTS = 100`，超过则丢弃最旧的。

### 3.4 回退文件：`fileHistoryRewind`

```ts
export async function fileHistoryRewind(updateFileHistoryState, messageId): Promise<void>
```

1. 在 `snapshots` 中找到 `messageId` 对应的快照（`findLast`，取最近一个）。
2. 调用 `applySnapshot(state, targetSnapshot)`。
3. `applySnapshot` 遍历所有 `trackedFiles`：
   - 如果目标快照中该文件的 `backupFileName` 为 `null` → 删除当前文件（如果存在）。
   - 如果目标快照中有备份文件名 → 通过 `checkOriginFileChanged` 比较当前文件与备份。
   - 只有确实不同才执行 `restoreBackup`，把备份 `copyFile` 回项目路径，并恢复权限。

### 3.5 变更检测

`checkOriginFileChanged` 优化了比较开销：

- 先比较文件是否存在。
- 再比较 `mode`、`size`。
- 如果当前文件 `mtimeMs < backup.mtimeMs`，直接认为没变。
- 否则才读取内容逐行比较。

这样既保证正确性，又避免每次快照都读大文件。

### 3.6 与 UI 的 diff 预览

`MessageSelector` 在列出消息时会异步调用 `fileHistoryGetDiffStats`，计算每个消息点相对于当前磁盘的：

- 会变更的文件数
- 增加/删除行数

这用的是 `diffLines` 对当前文件内容与目标备份做行级 diff。

## 4. 对话回退的实现：`rewindConversationTo`

在 `screens/REPL.tsx:3661`：

```ts
const rewindConversationTo = useCallback((message: UserMessage) => {
  const prev = messagesRef.current
  const messageIndex = prev.lastIndexOf(message)
  if (messageIndex === -1) return

  setMessages(prev.slice(0, messageIndex))          // 截断消息
  setConversationId(randomUUID())                   // 新 fork
  resetMicrocompactState()
  if (feature('CONTEXT_COLLAPSE')) resetContextCollapse()

  setAppState(prev => ({
    ...prev,
    toolPermissionContext: {
      ...prev.toolPermissionContext,
      mode: message.permissionMode ?? prev.toolPermissionContext.mode,
    },
    promptSuggestion: { ... } // 清空建议
  }))
}, ...)
```

关键点：

- 通过 `setMessages(prev.slice(0, messageIndex))` 把该消息及之后所有消息移除。
- 生成新的 `conversationId`，相当于创建了一个新的对话分支（fork）。
- 恢复该消息当时的权限模式。
- 清空 micro-compact 和 context-collapse 缓存，避免引用已被截断的 tool_use_id。

`restoreMessageSync` 还会把该用户消息的文本重新填到输入框，并把粘贴的图片恢复到 `pastedContents`。

## 5. 为什么 `/rewind` 能真正回退修改

`/rewind` 不是简单的“撤销最后一步”，它是一个基于**预写备份 + 按消息快照 + 文件系统恢复**的完整机制：

1. **修改前备份**：每次工具写文件前，都会先调用 `fileHistoryTrackEdit`，把当前内容保存到 `~/.claude/file-history/{sessionId}/` 下。这样即使后续写入失败或反复编辑，初始版本（v1）仍然安全。

2. **按消息建立快照**：每个用户消息处理完后，`fileHistoryMakeSnapshot` 会为所有被追踪文件建立一个状态点。这个快照用 `messageId` 作为键，因此用户可以选择“回退到某条消息之前”。

3. **备份与项目目录隔离**：备份存放在 Claude 配置目录，而不是项目内临时文件或 git 中，因此不受用户手动编辑、git 操作或 `.gitignore` 影响。

4. **恢复是文件系统级操作**：`applySnapshot` 通过 `copyFile` 把备份写回项目路径，`unlink` 删除快照中不存在的文件，并恢复文件权限。它能处理文件修改、新增、删除三种情况。

5. **会话持久化**：`recordFileHistorySnapshot` 把快照写入会话日志；`fileHistoryRestoreStateFromLog` 在恢复会话时重建 `FileHistoryState`；`copyFileHistoryForResume` 会把旧 session 的备份硬链接/复制到新 session，因此跨会话也能回退。

6. **变更检测避免误写**：恢复前会再次对比当前文件与备份，只有真正不同才写盘，减少不必要的文件抖动。

7. **对话与代码解耦**：用户可以选择只回退对话、只回退代码或两者都回退，互不影响。

## 6. 限制与注意事项

- 只追踪被 Claude 工具明确编辑过的文件；用户在编辑器里手动改的文件、或在 Bash 里通过未识别命令改的文件不会被自动追踪。
- 备份目录在 `~/.claude/file-history/{sessionId}`，会占用磁盘空间，但快照上限为 100，旧快照会被丢弃。
- 非交互式会话默认关闭，可通过 `CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING` 开启。
- 可通过 `CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING` 完全禁用。

## 7. 关键文件索引

| 文件 | 作用 |
|------|------|
| `src/commands/rewind/index.ts` | 命令注册 |
| `src/commands/rewind/rewind.ts` | 唤起 MessageSelector |
| `src/components/MessageSelector.tsx` | 回退选择 UI |
| `src/screens/REPL.tsx:3661` | 对话回退逻辑 |
| `src/utils/fileHistory.ts` | 文件备份、快照、恢复核心 |
| `src/utils/handlePromptSubmit.ts:528` | 每次用户消息后生成快照 |
| `src/tools/FileEditTool/FileEditTool.ts:435` | 编辑前追踪文件 |
| `src/tools/FileWriteTool/FileWriteTool.ts:259` | 写入前追踪文件 |
| `src/tools/BashTool/BashTool.tsx:393` | Bash 改文件前追踪 |
| `src/tools/BashTool/BashTool.tsx:627` | 执行模拟 `sed` 写入 |
| `src/tools/BashTool/sedEditParser.ts` | 解析/模拟 `sed -i` 命令 |
| `src/components/permissions/SedEditPermissionRequest/SedEditPermissionRequest.tsx` | `sed` 编辑权限预览与确认 |
| `src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx` | Bash 权限对话框分流到 sed 预览 |
| `src/utils/sessionStorage.ts:1476` | 持久化快照 |
| `src/utils/sessionRestore.ts:105` | 恢复会话时重建文件历史 |
