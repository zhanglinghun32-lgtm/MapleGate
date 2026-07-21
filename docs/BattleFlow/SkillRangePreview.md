# SkillRangePreview

## 核心設計

`SkillRangePreview` 不再放在 `ControlCharacterUI.ui`。它會生成為目前受控角色 Entity 的直接子 Entity，因此自然繼承角色的 Transform，角色移動時不需要 UI `OnUpdate` 追蹤。

```text
ControlledActor Entity
└─ SkillRangePreview Entity
   ├─ TransformComponent
   ├─ PolygonRendererComponent
   └─ SkillRangePreviewComponent
```

Polygon Points 使用以角色為原點的 local coordinates，`skillConfig` 的 range 數值直接使用 world units，不再轉換成 UI pixels。

## 所有權與生命週期

- `ControlCharacterUIComponent` 持有目前 Preview Entity 的 reference。
- 選取技能時呼叫 `EnsureSkillRangePreview()`；同一角色重用同一個 Preview。
- 更換技能只更新 Polygon Points。
- 取消、送出技能或離開輸入階段只呼叫 `Hide()`。
- 切換受控角色時先 Destroy 舊 Preview，再由新角色於選取技能時建立。
- UI `OnEndPlay` 呼叫 `ReleaseSkillRangePreview()`，避免留下孤兒 Entity。
- 角色被銷毀時，其直接子 Preview 也會失效；下一次 Ensure 會重新建立。

`Hide()` 與 `ReleaseSkillRangePreview()` 都必須可重複呼叫，讓取消、回合切換與 UI 關閉的清理順序保持安全。

## 責任邊界

- Preview 是 ClientOnly 操作提示，不是戰鬥權威狀態。
- Preview 不決定合法目標或命中結果；BattleSystem／SkillActor 仍負責實際解析。
- 角色中心與角色朝向型範圍使用角色子 Entity。
- 未來若技能以任意地面點為中心，應另外建立 map-root TargetPoint Preview。
