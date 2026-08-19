# 変更計画: 蓋をStep 0として扱えるようにする

現在の実装（`FindTarget`がActor経由、`HandleMontageBlendingOut`で無条件Destroy）を基準にした変更点。互換用のAPIは残さない。

## 設計原則

**RootもStepもすべて「Target Component＝Action＋Prop Mesh＋操作対象」。Sequence Componentは並べて順に再生するだけで、Actorの破棄などゲーム状態は変更しない。**

| # | 変更 | 解決する問題 |
|---|---|---|
| A | Step登録をActor単位 → Target Component単位 | 箱にRoot用と蓋用の2つのTarget Componentを置ける |
| B | Root Targetを明示参照にする | 箱に2つあるとき、どれがRootか曖昧にならない |
| C | 確定時の自動Destroyを廃止 | 蓋が消えない。消すかどうかはゲーム側の判断 |

---

## 1. `InteractionSequenceComponent.h`

### 1-1. Step登録API（A）

- 削除: `SetStepTargetActor(StepIndex, AActor*)`
- 追加: `SetStepTarget(StepIndex, UInteractionSequenceTargetComponent*)` — BlueprintCallable
- 追加: `GetStepTarget(StepIndex)` — BlueprintPure。未登録・範囲外はnullptr

### 1-2. Root Target参照（B）

- 追加: `FComponentReference RootTarget` — EditAnywhere、Component Picker、AllowedClassesはTarget Component
- 未指定ならOwner上で最初に見つかるTarget Componentを使う

### 1-3. 内部状態（A）

- 削除: `TArray<TObjectPtr<AActor>> StepTargetActors`、`GetStepTargetActor()`
- 追加: `TArray<TObjectPtr<UInteractionSequenceTargetComponent>> StepTargets`
- `FindTarget(StepIndex)` の宣言はそのまま

---

## 2. `InteractionSequenceComponent.cpp`

### 2-1. `FindTarget`（A・B）

```text
if StepIndex == INDEX_NONE:
    RootTarget が指すComponentがあればそれを返す
    なければ Owner の最初の Target Component を返す
else:
    StepTargets[StepIndex] が有効ならそれを返す、無効なら nullptr
```

### 2-2. `SetStepTarget` / `GetStepTarget`（A）

```text
SetStepTarget(StepIndex, Target):
    StepIndex が範囲外      → Warning ログ（"Set Step Count を先に呼ぶ"）して return
    Target == Root Target   → Error ログして return
    StepTargets を StepCount にリサイズして格納（nullptr で登録解除）

GetStepTarget(StepIndex):
    配列範囲内ならその要素、でなければ nullptr
```

### 2-3. `GetActiveTargetActor`（A）

```text
FindTarget(ActiveStepIndex) の Owner を返す。無効なら nullptr
```

### 2-4. `StartOrResume` の初期状態記録ループ（A）

```text
for StepIndex in [INDEX_NONE .. StepCount-1]:
    Target = FindTarget(StepIndex)
    Step かつ Target 無効 → continue   （確定後に破棄されたStep・未登録Stepは対象外）
    Target 無効 or CaptureInitialState 失敗 → Fail
```

判定は現状と同じ（Rootは必須、Stepは欠けていてもよい）。取得元が変わるだけ。

### 2-5. `HandleInput` の `StartNextStep`（A）

```text
TargetActor = GetStepTarget(StepIndex) の Owner（無効なら nullptr）
```

### 2-6. `HandleMontageBlendingOut`（C）

```text
現在:
    ++ProgressIndex
    ReleaseActiveMontages
    OnCommitted.Broadcast
    Target Actor を Destroy              ← 削除
    StepTargets[StepIndex] = nullptr     ← 削除
    PlayAction(Root, RootResumeSection)

変更後:
    ++ProgressIndex
    ReleaseActiveMontages
    OnCommitted.Broadcast                ← ゲーム側がここで Destroy するかどうか決める
    PlayAction(Root, RootResumeSection)
```

BP側でDestroyされれば`StepTargets[i]`は`IsValid`が偽になり、以後の`FindTarget`は自然にnullptrを返す。配列を消す必要はない。

### 2-7. 変更不要

`EndPlay`、`RestoreUncommittedTarget`、`ResetProgress` は`FindTarget`経由なのでそのまま。

### 2-8. エラーメッセージ

`PlayAction`の「Target Actor is missing or has no Target Component」→「Target Component is not registered or has been destroyed」

---

## 3. `InteractionSequenceTargetComponent.h / .cpp`

**変更なし。** クラスコメントだけ「1つのActorに複数追加可。Root用とStep用を同じActorに置ける」へ。

---

## 4. BP側の変更

### 4-1. 箱BP

```text
BP_Box
 ├ Interaction Sequence Component
 │    Root Target = BoxRootTarget
 │    Step Count  = 1 + アイテム数
 ├ BoxRootTarget（Target Component）
 │    Action   = 中身を見る（In → Loop → Out）
 │    Gates    = Loop/Confirm/StartNextStep、Loop/Cancel/Jump→Out
 │    Animated Skeletal Mesh = 箱本体（動くなら）
 └ BoxLidTarget（Target Component）← 追加
      Action   = 蓋を開ける（Character + 蓋Prop、Gateなし、再生し切って確定）
      Animated Skeletal Mesh = 蓋の Skeletal Mesh Component
```

### 4-2. 箱BPの `BeginPlay`

```text
Set Step Count( 1 + Items.Length )            ← 最初に
Set Step Target( 0, BoxLidTarget )             ← 蓋 = Step 0
For Each Items with Index:
    Spawn Item
    Item の Target Component に Set Action( Element.Action )
    Set Step Target( Index + 1, そのComponent )  ← +1
```

### 4-3. アイテムBP（C）

```text
On Committed → 取得処理 → Destroy Actor(Self)   ← 従来Componentがやっていた分
```

### 4-4. 蓋の開閉状態の保持

Step確定時に蓋のProp Montageは停止され、蓋MeshはAnimBPのベースポーズへ戻る。開いた状態を維持するのはAnimBPの責務。

```text
蓋AnimBP: IsOpen が true なら開いたポーズ
箱BP:
    BoxLidTarget.On Committed        → IsOpen = true
    BeginPlay / OnLoad 後             → Progress Index >= 1 なら IsOpen = true
    Reset イベント                    → IsOpen = false
```

### 4-5. ProgressIndexの読み替え

| ProgressIndex | 意味 |
|---:|---|
| `INDEX_NONE` | 未開始 |
| `0` | 中身を見た。蓋は閉じている |
| `1` | 蓋が開いた。アイテム0個 |
| `k` | アイテム `k-1` 個取得 |
| `StepCount + 1` | 完了 |
