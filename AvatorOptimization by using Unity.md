# VRChat アバター軽量化ノウハウ

**Claude Code + MCP for Unity / Unity上での非破壊最適化**

## この文書の目的

この文書は、Claude Code + MCP for Unity を利用して、VRChat PCアバターをUnity上で可能な限り軽量化するための実践的な知識ベースである。

過去のアバター軽量化で得られた実測知見と、Avatar Optimizer（AAO）、TexTransTool（TTT）、Meshia、NDMFなどを利用する最適化手法を統合している。

**最重要方針は、見た目・揺れ・機能を維持したまま、検証可能な非破壊最適化を自律的に実行すること。**

安全性を十分確認できる処理まで逐一ユーザーへ確認する必要はない。一方、機能や外観への影響が不明な処理、あるいは意図的に機能を削る処理は、ユーザーに提案して許可を得る。

本書では、アバター固有の実測結果を一般則と混同しない。各ツールの機能はバージョンによって変化するため、実際に導入されているバージョンとビルド結果を優先する。

---

# 0. 最上位ルール

## 0.1 元シーンを絶対に変更しない

最適化を始める前にUnityシーンファイルを複製する。

例:

- `Assets/AvatarOriginal.unity` — 元シーン。変更・保存禁止
- `Assets/AvatarOriginal_lightweight.unity` — 軽量化作業用

以後の編集と保存は、原則として作業シーンのみで行う。

元シーンは、最適化前の測定、外観比較、PhysBone比較などの参照用として使用する。

比較のためにGameObjectを追加複製してもよいが、基本の保護単位はシーン複製とする。

## 0.2 元アセットも変更しない

シーンを複製しても、以下のアセットは共有されている可能性がある。

- Material
- Texture
- Mesh
- Animator Controller / FX Controller
- Animation Clip
- Prefab
- ScriptableObject

共有アセットを直接編集すると、元シーンへ影響する場合がある。

直接変更が必要ならアセットを複製し、作業側だけへ割り当てる。

ただし、AAOやTTTなどのNDMF対応ツールでビルド時に非破壊処理できるなら、原則としてそちらを優先する。

## 0.3 基本目標

優先順位は以下。

1. 見た目を維持する。
2. PhysBoneの揺れを維持する。
3. Grab、Contact、Constraintなどの動作を維持する。
4. Expression Menu、着せ替え、Toggle、ギミックを維持する。
5. 元シーン・元アセットを変更しない。
6. 上記を守った範囲でPerformanceを最大限改善する。

数値だけ改善しても、見た目や機能が意図せず変化した場合は成功としない。

## 0.4 「非破壊」と「無影響」を区別する

非破壊とは、元データを変更せず、設定を取り消すことで元の状態へ戻せることを意味する。

非破壊であっても、最終ビルドの見た目や機能が変わる可能性はある。

したがって、

**非破壊処理であること + 影響がないことを十分検証できること**

を、自動実行・自動採用の基本条件とする。

非破壊ツールによる処理は、問題が見つかった場合にロールバックできるようにする。

## 0.5 自動実行とユーザー確認の境界

### A. 無影響を十分検証できる処理

ユーザーへの事前確認なしで実行してよい。

例:

- AAO Trace and Optimize
- AAO Max Texture Size
- 安全なMaterial / Renderer統合
- 検証可能なMeshiaによる軽量化
- TTTによる非破壊アトラス化
- 明らかに不要なGameObjectのEditorOnly化
- AAOによる非破壊的な隠れポリゴン削除
- 完全互換で機能差がないPhysBone統合

ただし、実行後の測定・比較は必要。

### B. 無影響を確定できない処理

効果と懸念点をユーザーに提示し、判断を求める。

例:

- 一部の衣装状態で見える可能性がある下着
- Toggle参照の有無を完全に追跡できないGameObject
- Grab挙動が変わる可能性のあるPhysBone統合
- 見た目の差が小さいが完全には一致しないMaterial統合

### C. 明確な機能削減・動作変更

ユーザーの事前許可が必要。

例:

- PhysBone自体の削除
- 揺れを変える関節間引き
- FX LayerやMenu機能の削除
- 使用可能な衣装・アクセサリーの削除
- Particle、Light、Contactなどの機能簡略化
- TransparentからCutoutへの変更など、外観を意図的に変更する処理

---

# 1. 推奨ツール

## 1.1 基盤

- VRChat SDK — Base / Avatars
- NDMF
- Modular Avatar

## 1.2 最適化

- Avatar Optimizer（AAO）
  - Trace and Optimize
  - Merge PhysBone
  - Merge Skinned Mesh
  - Merge Material
  - Max Texture Size
  - Remove Mesh系
  - Merge Bone
  - Replace End Bone With Endpoint Position
  - BlendShape最適化
- TexTransTool（TTT）
  - AtlasTextureなど
- Meshia Mesh Simplification
- lilNDMFMeshSimplifier

## 1.3 測定・検証

- anatawa12's gists — **Actual Performance Window**
- lilAvatarUtils
- Gesture Manager
- MotionProbe、または同等のPhysBone時系列比較手段
- 同一条件での画像比較スクリプト

## 1.4 AI・自動化

- MCP for Unity
- Unity Editorの一時検証スクリプト
- 必要に応じてRoslyn等のC#実行環境

ツールの仕様・対応Shader・処理順・バグはバージョン依存である。実作業では導入バージョンを記録する。

---

# 2. 最適化前の基準状態を測定・記録する

## 2.1 基準状態とは

最適化前の状態を記録し、最適化後の結果と比較するための基準とする。

本書では原則として **「最適化前の基準状態」** と表記する。

この記録がないと、軽量化による数値改善や外観・動作の変化を客観的に判定できない。

## 2.2 Performanceの記録

Actual Performance Windowを使用して、少なくとも以下を記録する。

- 総合Performance Rank
- 各Performance項目の測定値とRank
- 現在のボトルネック項目
- 最適化対象になりそうなRenderer / Material / PhysBone

必要に応じて補助スクリプトで内訳を取得する。

ただし、独自スクリプトの計算値だけで最終Rankを判断しない。

## 2.3 外観の記録

比較用画像を撮影する。

推奨視点:

- 正面
- 側面
- 背面
- 顔アップ
- 足元
- 頭部背面
- 透明素材のある部位
- 衣装の特徴的な部位

必要に応じて重要なBlendShapeやMenu切り替え状態も記録する。

## 2.4 PhysBoneの記録

PhysBoneを変更する予定がある場合、最適化前にMotionProbeなどで基準動作を記録する。

記録対象:

- チェーンごとの時系列変位
- 最大変位
- 揺れのピーク
- 比較対象ボーン
- 測定シナリオ

Grabを使う部位は、Grab操作も別途確認する。

## 2.5 機能の記録

以下のうち、対象アバターで利用されているものを記録する。

- Expression Menu
- Toggle
- 着せ替え
- BlendShape
- Grab
- Contact
- Constraint
- Particle
- Light
- ギミック

全組み合わせを無条件に網羅する必要はないが、最適化対象に関係する状態は漏れなく確認する。

---

# 3. Performance測定の基本方針

## 3.1 Actual Performanceを正式評価とする

VRChatのPerformance Rank判定には、anatawa12's gistsのActual Performance Windowを使用する。

特にAAO、TTT、Modular Avatarなどの非破壊処理は、Editor上のHierarchyと最終ビルド状態が異なる。

そのため、編集状態のComponent数やTexture数だけで最終Rankを推定しない。

Play Modeとアップロード時のビルド処理が完全に一致しない環境もある。

最終評価は、実際のビルド処理を反映した測定結果を優先する。

## 3.2 Rankの固定閾値は本書に記載しない

Performance Rankの数値閾値はVRChat SDKや公式資料で確認できるため、本書では固定表を持たない。

閾値が変更されても知識ベースが古くならないよう、Actual Performanceの表示を正とする。

## 3.3 ボトルネックを優先する

Performance Rankは基本的に最も悪いカテゴリに支配される。

したがって、すでに十分良いカテゴリをさらに削減するより、現在のボトルネックを改善する方がRank向上には有効。

例:

- Texture Memoryがボトルネック → Texture Size / Atlas
- Material Slotsがボトルネック → Material / Renderer統合
- Trianglesがボトルネック → Mesh Simplification / Hidden Mesh
- PhysBone Componentsがボトルネック → 安全なMerge
- Collision Checksがボトルネック → Colliderと対象Transformの関係を調査

## 3.4 Performance Rankと実負荷を区別する

Rankが改善しても、GPU・CPU負荷が必ず同じ割合で減るわけではない。

Rankだけでは十分に表現されない要素:

- Shader Pass数
- Shaderの複雑さ
- Transparency / Overdraw
- Draw Call
- Animator内部構造
- BlendShapeの実行コスト
- Particleの画面占有率

Rankを改善すると同時に、これらを悪化させない。

---

# 4. 標準ワークフロー

## Phase 1 — 保護と測定

1. 元シーンを複製する。
2. 作業シーンを開く。
3. 元シーンと共有アセットが保護されていることを確認する。
4. 最適化前のActual Performanceを記録する。
5. 比較用画像を撮影する。
6. 必要ならMotionProbeを取得する。
7. 重要なMenu・Toggle・機能状態を記録する。
8. ボトルネックを特定する。

## Phase 2 — 自動で実施できる非破壊最適化

優先候補:

1. AAO Trace and Optimize
2. 明らかに不要なObjectのEditorOnly化
3. AAO Max Texture Size
4. TTT AtlasTexture
5. AAO Material / Renderer統合
6. 安全なPhysBone統合
7. Meshia等による段階的なMesh簡略化
8. AAOによる隠れポリゴンの非破壊削除

この順序は固定ではない。

実際には、現在のボトルネック、ツールの処理順、最適化同士の依存関係に応じて変更する。

## Phase 3 — 各処理の検証

原則として次の単位で実施する。

**変更前測定 → 1種類の最適化 → Actual Performance → 画像比較 → 必要なら動作比較 → 採用またはロールバック**

複数の大きな最適化を同時に入れて、問題の原因を追跡できなくしない。

## Phase 4 — 追加提案

安全な最適化だけで目標に届かない場合は、機能削減・簡略化の候補を提示する。

例:

- 使用可能だが普段使わない衣装の除去
- FX Layer削除
- PhysBone関節の間引き
- 揺れの削除
- Particle簡略化
- Light削除
- Shader表現の簡略化

影響と改善見込みを説明し、許可されたものだけ実行する。

## Phase 5 — 最終検証

- Actual Performance
- 外観比較
- MotionProbe
- Grab / Contact / Constraint
- Menu / Toggle / BlendShape
- 元シーンと元アセットの保護
- 作業シーンの保存状態
- アップロード先の確認

---

# 5. AAO Trace and Optimize

Trace and Optimizeは最初に試す自動最適化の第一候補。

バージョンや設定に応じて、次のような最適化を行う。

- 不要なGameObject・Componentの整理
- 不要なBoneの削除
- BlendShape最適化
- Skinned Mesh Renderer統合
- Material Slot整理
- PhysBoneの自動最適化
- PhysBone Collider統合
- Endpoint最適化
- Animator最適化
- Texture関連の最適化

ただし、すべての処理が常に実行されるわけではない。

Avatarの構成、Animation参照、Shader情報、Grabの設定、他ツールとの組み合わせによって、実行可能な最適化が変わる。

**手動最適化を追加する前に、Trace and Optimizeがすでに実施していないか確認する。**

Optimization MetricsやBuild Logを確認し、Actual Performanceで最終結果を測定する。

---

# 6. EditorOnlyによる不要Objectの除去

## 6.1 基本方針

**最終ビルドで不要であり、見た目・揺れ・機能に影響しないことを十分確認できるGameObjectは、ユーザー許可なしでEditorOnly化してよい。**

EditorOnlyは元のMeshやTextureを直接破壊する処理ではなく、作業シーン上で取り消し可能な最適化として扱う。

ただし、EditorOnly化されたObjectは最終ビルドから除外されるため、機能に影響する可能性がある場合は勝手に実施しない。

## 6.2 特に有力な候補

- 服の下に完全に隠れている下着
- 現在のAvatar構成では使われない衣装
- 使われない装飾品
- 使われないRenderer
- 古い改変の残骸
- ビルド時に参照されない補助Object

下着は、Toggle等がなく、どの使用可能な衣装状態でも完全に隠れているなら、非常に有力な候補になる。

## 6.3 自動EditorOnly化の確認条件

以下を調査する。

### 表示・外観

- 現在の状態で見えない
- 使用可能な衣装・Menu状態でも見えない
- Animationによって表示されない
- BlendShapeによって露出しない
- 複数方向からの画像比較で差分がない

### 参照関係

- FX / Animatorから参照されない
- Expression Menuから利用されない
- Modular Avatarから必要とされない
- Animation Clipからactive状態等を変更されない
- 他のギミックから参照されない

### 動作・依存関係

- PhysBoneの必要な対象でない
- Contact / Constraintの依存先でない
- 他Rendererが必要とするBoneを持たない
- 必要なSkinned Meshのweightに関係しない
- 削除により他の機能が失われない

### 最終確認

- EditorOnly化後のビルド結果を測定する
- 外観・機能を比較する
- 問題があれば元へ戻す

## 6.4 下着の例

次の条件を満たす下着Rendererを想定する。

- デフォルトで服の下に隠れている
- 下着表示Toggleがない
- 衣装を脱ぐ機能がない
- FXやAnimationから参照されない
- 他の必要なComponentから参照されない
- どの関連表示状態でも見えない

この場合は自動EditorOnly化してよい。

効果として、構成に応じて以下が削減され得る。

- Triangles
- Skinned Mesh Renderer
- Material Slots
- Texture Memory
- Bones
- PhysBone

ただし、TextureやBoneが他のRendererと共有されている場合は、それらまで必ず削減されるわけではない。

## 6.5 不確実ならユーザーへ提案する

例:

「下着Rendererは通常状態で完全に隠れています。現在確認できるToggleからの参照はありませんが、一部の衣装状態で露出する可能性を排除できません。EditorOnly化するとRendererとMaterial Slotを削減できます。実施しますか？」

**inactiveであることだけを理由に不要と判断しない。**

また、`GameObject.Find`ではinactive rootを取得できないため、必要に応じて`Scene.GetRootGameObjects()`から探索する。

---

# 7. AAOによる非破壊的な隠れポリゴン削除

## 7.1 基本方針

**AAOによるビルド時の非破壊的な隠れポリゴン削除は、影響がないことを十分確認できればユーザー許可なしで実行してよい。**

対象となる機能の例:

- Remove Mesh By BlendShape
- Remove Mesh By Box
- Remove Mesh By Mask
- Remove Mesh By UV Tile

これらは元Meshを直接編集せず、ビルド時に不要なGeometryを除去するための手段である。

## 7.2 自動実行の条件

- 対象が最終的に不要なGeometryである
- 関連する表示状態で見えない
- BlendShapeで露出しない
- 衣装Toggle等で露出しない
- 揺れや姿勢変化で露出しない
- 削除後の画像比較で問題がない
- 問題があれば簡単に元へ戻せる

非破壊だからといって、検証なしに採用しない。

## 7.3 隠れポリゴン削除の注意点

以前の実測では、衣装の下にあると判断して膝・脛・スパッツ付近のGeometryを削減したところ、実際には見えている部分があり、穴が生じた。

したがって、正面画像だけで判断しない。

必要に応じて、

- 水平8方向
- 上方4方向
- 下方・特殊角度2方向

など、多方向から比較する。

## 7.4 Renderer単位のEditorOnlyを優先する場合

対象Renderer全体が不要なら、まずEditorOnly化を検討する。

Renderer全体を除外できれば、GeometryだけでなくMaterial Slotなども削減できる可能性がある。

一方、Rendererの一部分だけが隠れている場合はAAO Remove Mesh系を使う。

つまり、

- Renderer全体が不要 → EditorOnly
- Rendererの一部が不要 → AAO Remove Mesh
- 不要かどうか不明 → 調査・比較・必要ならユーザー確認

と判断する。

---

# 8. Texture Memory最適化

## 8.1 AAO Max Texture Size

大きなTextureはTexture Memoryの主要なボトルネックになりやすい。

AAO Max Texture Sizeでビルド時の最大Texture Sizeを制限する。

まずは4Kから2Kへの変更などを候補にする。

ただし一律に縮小せず、必要に応じてTextureごとに調整する。

比較対象:

- 顔
- 目
- 衣装
- Normal Map
- 細かい模様
- 文字・ロゴ
- 透明素材

## 8.2 過去の実測例

あるアバターでは4Kから2Kへの制限により、Texture Memoryが約164MBから76MBまで減少した。

通常の視点では差が非常に小さかったが、Normal Mapの細かい織り目がやや弱くなった。

これは参考例であり、別のアバターでも同じ削減量になるとは限らない。

## 8.3 Mipmap・圧縮形式

Max Texture Sizeでは、MipmapがないTextureや一部の圧縮形式に制約がある。

警告が出たTextureを確認する。

元Texture Import Settingsを直接変更しない。

必要な場合はTextureを複製し、作業側だけへ割り当てる。

## 8.4 Editorのメモリ値とVRChatの値

Unity Profiler等のTexture Memory表示と、VRChatのPerformance評価に使われるTexture Memoryは必ずしも一致しない。

正式な比較はActual Performanceで行う。

---

# 9. 非破壊Texture Atlas

## 9.1 AtlasはUnity上でも非破壊で実現できる

Texture AtlasはBlender等で元MeshやUVを直接編集しなければ実現できない処理ではない。

TexTransToolのAtlasTexture等を使えば、NDMFによるビルド時の非破壊処理として実施できる。

したがって、Unity上の最適化候補に含める。

## 9.2 主な目的

- Texture Memory削減
- 使用Texture領域の整理
- Material統合の補助
- Material Slot削減の下準備

ただし、Atlas化だけで必ずTexture Memoryが減るとは限らない。

Atlasのサイズや配置によっては、逆に増える場合もある。

## 9.3 Material Slot削減との関係

Material Assetを共通化するだけでは、RendererのMaterial Slotが減るとは限らない。

Material SlotやSubMeshを削減するには、AAOのMerge Skinned MeshやMaterial Slot統合等と組み合わせることを検討する。

推奨候補:

**TTT AtlasTexture + AAO Trace and Optimize / Merge Skinned Mesh**

実際にSlotが減ったかはActual Performanceで確認する。

## 9.4 Atlas後の検証

- Texture Memory
- Material Slots
- Shader設定
- Normal Map
- Emission
- MatCap
- Mask
- UV
- Texture seam
- Mipmap
- Transparency
- Material Swap Animation
- 非表示衣装の表示状態

特に、通常は非表示だがMenuから表示されるRendererをAtlas処理から漏らさない。

Shaderによって対応Textureや変換方法が異なるため、未知Shaderは慎重に扱う。

---

# 10. Material / Renderer最適化

## 10.1 Material Slot

Material SlotはMaterial Assetの個数とは異なる。

同じMaterialを複数Slotに設定していても、Slot数が多いままの場合がある。

したがって、Materialの共通化だけでなく、RendererやSubMeshの統合を検討する。

## 10.2 AAO Merge Skinned Mesh

統合可能なSkinned Mesh RendererはAAOで統合する。

期待できる効果:

- Renderer数削減
- Material Slot削減
- Draw Call削減
- Skinning関連の処理整理

ただし、Rendererの独立ON/OFF、BlendShape、Material Swap、Transparencyなどに注意する。

## 10.3 Materialの設定差

同じShaderを使っていても、Materialが同一とは限らない。

過去の実測では、同じShaderのMaterial同士でも、影・Rim・Outline・MatCapなど多数のPropertyが異なっていた。

Shader名だけで統合を判断しない。

見た目が変わらない統合は自動候補にできるが、質感を変更する統合はユーザー確認が必要。

## 10.4 Shader負荷

Performance RankだけではShaderの負荷を十分に評価できない。

確認対象:

- Pass数
- Transparent / Cutout / Opaque
- Overdraw
- Tessellation
- Outline
- 複雑なScreen Space Effect

TransparentをCutoutへ変更する等の処理は、見た目が変わるため自動実行しない。

---

# 11. Mesh Simplification

## 11.1 基本方針

MeshiaまたはlilNDMFMeshSimplifier等を使用し、大きなMeshから段階的に削減する。

- Renderer単位で設定する
- Border Edgeを保護する
- 一度に大きく削らない
- 各段階で画像比較する
- 顔は特に保守的に扱う
- 細いGeometryは慎重に扱う

## 11.2 過去の実測例

あるアバターでは、

- Shoes: 0.55
- Hoodie: 0.65
- Shirt: 0.80

程度の設定が成立した。

一方、Shoesを0.45まで削減すると黒いArtifactが発生した。

Barycentric InterpolationでArtifactが改善しても、布の細かい模様がぼやけた例もある。

このため、設定値を別アバターへそのまま適用しない。

## 11.3 注意すべきGeometry

- 顔
- 眼鏡の細いフレーム
- リボン
- レース
- まつ毛
- 細いワイヤー
- 透明Mesh
- BlendShapeで変形するMesh

## 11.4 BlendShapeで出現するゼロ面積ポリゴン

通常状態では面積0に潰れているが、BlendShapeによって現れるGeometryがある。

通常状態だけを基準にMesh Simplificationすると、そのGeometryが失われる可能性がある。

削減前に、

- 通常状態で面積0のTriangle
- BlendShape適用時に面積が生じるTriangle

を調査する。

該当するMeshは削減対象から外すか、BlendShape状態を含めて十分に比較する。

---

# 12. PhysBone最適化

PhysBoneは以下を別々に確認する。

- PhysBone Component数
- Affected Transform数
- Collider数
- Collision Check数

## 12.1 まず自動最適化を確認する

AAO Trace and Optimizeによって、未使用PhysBoneや統合可能なColliderなどが最適化される場合がある。

手動処理の前にビルド後の値を確認する。

## 12.2 Merge PhysBoneの安全条件

安全な統合候補は、原則として以下が一致するもの。

- 用途
- rootの親関係
- PhysBone設定
- Curve
- Collider
- Endpoint条件
- Chain長
- Bone数
- Grab等のInteraction条件

SerializedObjectでPropertyを比較すると見落としが少ない。

ただしrootTransform等、チェーンごとに異なることが当然のPropertyは適切に除外する。

## 12.3 Chain長が異なる場合

**同じChain長同士で統合することを原則とする。**

異なるChain長でも差が小さい例はあったが、別の例では約1.5cmのMotion差が発生した。

異なる長さの統合が必要ならMotionProbeで比較する。

差が大きければグループを分割する。

## 12.4 Endpoint

Merge PhysBone後は末端Boneの扱いに注意する。

確認事項:

- Leafの方向
- Leafの長さ
- Endpoint Position
- Merge後のLeaf構成

近い長さ・同方向のLeafは比較的統合しやすい。

長さの外れ値がある場合は別グループにする。

AAOのバージョンによって、Merge PhysBone側のOverrideやReplace End Bone With Endpoint Positionを利用する場合がある。

特定の方法に固定せず、最終Motionで判断する。

## 12.5 Grab挙動

PhysBoneをMergeすると、通常の揺れが同じでも独立Grabできる箇所が減る場合がある。

したがって、Grab可能PhysBoneのMergeは単なる無影響最適化として扱わない。

Grab機能が変わる可能性があればユーザーへ提案する。

## 12.6 Radius等のOverride

RadiusなどInteractionに関係する値を変更して統合できる場合がある。

しかし、通常Motionが同じでも接触時の挙動が変わる可能性がある。

値の変更が必要なら、Interactionへの影響を確認する。

## 12.7 Merge Bone

BoneやPhysBone Transformを減らせるが、揺れが変化する可能性がある。

原則としてユーザー許可が必要。

過去には、

- 2関節程度の短いChainを1本化すると揺れが大きく減少
- 長いChainの間引きで揺れ幅が約半分
- Angle Limitのある根元への統合でほとんど動かなくなる

という例があった。

Boneが短いという理由だけで安全と判断しない。

## 12.8 PhysBone自体の削除

揺れを失うため、ユーザー許可が必要。

候補を探す場合は、MotionProbeで揺れ幅が小さい部位や通常見えない部位を調査する。

ただし、自動最適化で最終ビルドから不要Componentが除去される場合は、その処理の無影響性を検証する。

## 12.9 PrefabSafeSetの注意

過去に、AAOのPrefabSafeSet系APIで1要素をRemoveしたつもりがSet全体が消えた例がある。

安全策:

- Removeを安易に信用しない
- Clear後に期待する要素を再構築する
- 必要ならSerializedObjectで編集する
- 変更後に要素数と内容を確認する

---

# 13. PhysBoneのMotion比較

## 13.1 標準はMotionProbe

同じMotion ScenarioをOriginalとOptimizedに与え、時系列で比較する。

記録項目:

- Max Difference
- RMS Difference
- Peak Displacement
- 消えたChain
- 対応付けできなかったChain

MergeによってBone名やChain構造が変わる場合があるため、適切に対応付ける。

## 13.2 Gesture Manager

Gesture ManagerのAttach状態が異なると比較条件が変わる場合がある。

OriginalとOptimizedで同じ条件を使う。

シーン複製方式では別Runとして測定しやすい。

## 13.3 補助的な角度比較

MotionProbeが使えない場合、共通Boneの方向ベクトルを記録し、Head Local空間などで角度変化を比較できる。

振り向き・走行・急停止など、再現可能なシナリオを用いる。

Endpoint置換でLeaf構成が変わる場合は、末端Boneそのものを比較基準にしない。

測定ノイズを確認するため、Original同士のControl Runも取得する。

---

# 14. BlendShape最適化

AAOによるBlendShape最適化を優先する。

候補:

- 未使用BlendShapeの除去
- 固定値BlendShapeのFreeze
- 統合可能なBlendShapeの整理

ただし、以下から参照される可能性を調査する。

- Expression Menu
- FX Controller
- LipSync
- MMD World
- Modular Avatar
- その他Animation

BlendShape名がMerge後に変更される場合は、比較スクリプトでも対応付けを行う。

---

# 15. Animator / FX最適化

## 15.1 AAO Animator Optimizer

Animatorの内部構造を最適化できる場合がある。

例:

- 冗長なTransitionの整理
- 不要なLayerの整理
- BlendTree等への変換・統合

具体的な最適化内容はAAOのバージョンによる。

Animator Component数だけでなく、内部の実行負荷にも注意する。

## 15.2 未使用FX Layer

使わない衣装ToggleやギミックのLayerを削除すると、Animator処理の削減やAAOの不要Object判定改善につながる場合がある。

ただし機能削除になるためユーザー許可が必要。

実施する場合は、

1. 元Controllerを複製する。
2. 作業用Controllerを編集する。
3. 初期状態を確認する。
4. Menu・Animation動作を比較する。
5. Actual Performanceを再測定する。

---

# 16. Contacts

Contactsは独立したPerformance項目として確認する。

まず次を分類する。

- 他人とのInteractionに必要
- Local用途だけでよい
- 未使用

Local Only化によってPerformance評価上のContact数を減らせる場合がある。

ただし、他人とのInteractionが失われる可能性があるため、機能変更が発生するならユーザーへ確認する。

未使用Contactの削除も、参照・機能を確認してから判断する。

---

# 17. Constraints

ConstraintはCountとDepthを分けて確認する。

## 17.1 Count

Constraint Componentの総数を調べる。

不要なConstraintや重複したConstraintを整理できないか確認する。

## 17.2 Depth

Constraint同士の依存関係が直列に連なるとDepthが増える。

単純なComponent数削減だけでなく、依存Chainを短くできないか調査する。

## 17.3 VRC Constraint

Unity ConstraintとVRC Constraintの違いに注意する。

SDKの変換・最適化機能を利用できる場合は候補とするが、変換後の動作を必ず比較する。

---

# 18. Expression Parameters

Parameterの容量・同期設定はPerformance Rankとは別の観点でも重要。

Local用途しかないParameterがSyncedになっていないか確認する。

ただし、同期を外すことで他人から見えるギミックが変わる場合がある。

機能変更になる場合はユーザーへ確認する。

---

# 19. Particle / Trail / Line

確認項目:

- Particle System数
- maxParticles
- Emission
- Mesh ParticleのTriangle数
- Trails
- Collision
- Renderer Material
- Transparency
- 画面占有率

最適化候補:

- maxParticles削減
- Emission削減
- Mesh Particleの軽量化
- Trails無効化
- Collision無効化
- Particle System統合

これらは見た目や機能を変更しやすいため、原則としてユーザーへ提案する。

---

# 20. Lights

Avatar上のRealtime LightはPerformance Rankだけでなく実際の描画負荷にも影響する。

調査項目:

- Lightの有無
- 常時ONか
- Shadowの有無
- Particle Lightか
- 実際に必要な演出か

削除やShadow無効化は見た目を変更する可能性があるため、ユーザー許可対象とする。

---

# 21. NDMFとビルド処理

NDMFでは、概ね次のようなBuild Phaseを経てAvatarが処理される。

- Resolving
- Generating
- Transforming
- Optimizing

Modular Avatar、AAO、TTTなどの処理が最終的にどの状態へ適用されるかを意識する。

Edit ModeのHierarchyだけで最終ビルド状態を推定しない。

問題が発生した場合は、

- Build Log
- NDMF Error Report
- Plugin Version
- 処理Phase
- PreviewとBuildの差

を確認する。

---

# 22. 外観比較

## 22.1 同一条件で撮影する

OriginalとOptimizedを、

- 同一Camera
- 同一姿勢
- 同一画角
- 同一照明
- 同一Animation状態
- 同一撮影タイミング

で撮影する。

比較画像は、

**Original / Optimized / Difference**

の3列などに整理する。

## 22.2 一時Cameraを使う

SceneView Cameraで安定した比較ができない場合は、一時CameraとRenderTextureを使用する。

画像はAssets外の一時ディレクトリに保存すると不要なImportを避けられる。

## 22.3 Animationによる差分

Idle、Breathing、Blinkなどで時間変化が発生する。

撮影タイミングが異なると、最適化とは無関係な差分が出る。

必要ならOriginalを複数回撮影し、自然な変動幅を確認する。

## 22.4 比較すべき状態

- 正面・側面・背面
- 顔アップ
- 足元
- 透明素材
- 重要BlendShape
- Menu切り替え
- 衣装状態
- PhysBoneで大きく動く状態

---

# 23. MCP for Unity / execute_codeの注意

## 23.1 C#コンパイラ差

環境によってexecute_codeで利用できるC#構文が異なる。

安全策:

- `UnityEngine.Object`を明示する
- Lambdaの変数名衝突を避ける
- 必要ならLINQを明示的に呼ぶ
- 新しい構文へ過度に依存しない

## 23.2 Domain Reload

Play開始などでDomain Reloadが発生すると、一時的なデータやCallbackが失われる場合がある。

長時間の測定データはファイルへ保存する。

## 23.3 EditorApplication.update

登録したCallbackは終了時に必ず解除する。

Play停止時にも自己解除できるようにする。

## 23.4 Active状態の復元

測定・撮影のためにGameObjectを一時的に非表示にした場合、元のActive状態を記録して復元する。

元から非表示だったObjectを表示状態で保存しない。

## 23.5 Dirty Scene

Playや自動処理後にSceneがDirtyになる場合がある。

意図しない変更を元シーンへ保存しない。

必要ならSceneを開き直して状態を確認する。

## 23.6 パス

ShellのCurrent Directoryが後続処理へ影響する場合がある。

可能な限りAbsolute Pathを使用する。

---

# 24. 最適化ごとの許可ポリシー

| 処理 | 原則 |
|---|---|
| Scene複製 | 自動実行 |
| 最適化前の測定・記録 | 自動実行 |
| Actual Performance測定 | 自動実行 |
| AAO Trace and Optimize | 自動実行・検証 |
| AAO Max Texture Size | 自動実行・画像比較 |
| TTT AtlasTexture | 自動実行・画像比較 |
| 安全なRenderer / Material統合 | 自動実行・画像比較 |
| Meshiaによる非破壊簡略化 | 自動実行・画像比較 |
| 無影響を確認できるEditorOnly | **自動実行可** |
| 無影響を確認できるAAO Remove Mesh | **自動実行可** |
| 完全互換のPhysBone Merge | MotionProbe等で検証できれば自動実行可 |
| Grab挙動が変わるPhysBone Merge | 事前提案 |
| 異なるChain長のPhysBone Merge | 原則回避・慎重に検証 |
| PhysBone Merge Bone | 事前許可 |
| PhysBone自体の削除 | 事前許可 |
| 使用可能な衣装の削除 | 事前許可 |
| FX Layer / Menu機能削除 | 事前許可 |
| Contactの機能変更 | 事前許可 |
| Particle簡略化 | 事前許可 |
| Light削除 | 事前許可 |
| Shader表現変更 | 事前許可 |
| 元アセットの直接破壊 | 原則禁止 |

**重要:** 「自動実行可」は「検証不要」を意味しない。比較して問題があればロールバックする。

---

# 25. ユーザーへの提案形式

機能変更や影響不明の処理を提案する場合は、次の情報を提示する。

**候補**

対象GameObject / Componentと処理内容。

**期待効果**

改善が見込まれるPerformance項目と、可能なら予測削減量。

**根拠**

- 現在の表示状態
- Animation参照
- Menu参照
- Component依存関係
- 画像比較結果
- Motion比較結果

**懸念点**

失われる可能性がある見た目・揺れ・機能。

**実行判断**

ユーザー許可待ち。

推定値は推定値として明示し、実測値と混同しない。

---

# 26. 変更後チェックリスト

## Performance

- [ ] Actual Performanceを再測定した
- [ ] 改善した項目を記録した
- [ ] 悪化した項目がない
- [ ] 現在のボトルネックを確認した
- [ ] 変更前後の値を比較した

## Visual

- [ ] 正面
- [ ] 側面
- [ ] 背面
- [ ] 顔アップ
- [ ] 足元
- [ ] 透明素材
- [ ] Normal Map
- [ ] Atlas Seam
- [ ] 重要BlendShape
- [ ] 関連するMenu / Toggle状態

## Motion / Function

- [ ] PhysBone変更時はMotionProbeを実施した
- [ ] Grab挙動を確認した
- [ ] Contactを確認した
- [ ] Constraintを確認した
- [ ] 必要なギミックが動作する

## Safety

- [ ] 元シーンを変更していない
- [ ] 元共有アセットを変更していない
- [ ] 意図しない機能削減がない
- [ ] 無影響と判断した根拠がある
- [ ] 問題があった処理をロールバックした
- [ ] 作業シーンのみ保存した

---

# 27. やってはいけないこと

- 元シーンを直接編集する
- 元アセットを無断で変更する
- 独自計算だけでPerformance Rankを断定する
- 複数の大きな変更を一度に入れる
- inactiveという理由だけでEditorOnly化する
- 非破壊だから無影響だと決めつける
- 隠れポリゴンを1方向だけ見て削除する
- Toggleや衣装状態を無視して隠れMeshを削除する
- Shader名だけでMaterialを統合する
- Atlas化しただけでMaterial Slotが減ったと判断する
- Atlas後のTexture Memoryを測定しない
- 異なるPhysBone Chainを検証なしでMergeする
- Grab挙動を無視してPhysBone Mergeする
- Zero-Area Triangleを無条件に削除する
- 顔を機械的に強く簡略化する
- PhysBoneの揺れをユーザー許可なしで削減する
- 使用可能なMenu機能を勝手に削除する
- Play Modeの値だけで最終ビルドを断定する
- 古いツール仕様を現在も正しいと決めつける

---

# 28. 推奨意思決定フロー

**START**

1. 元シーンを複製する。
2. 最適化前の基準状態を測定・記録する。
3. Actual Performanceでボトルネックを特定する。
4. AAO Trace and Optimizeを確認する。
5. ボトルネックに対応する非破壊最適化を検討する。
6. 無影響を十分確認できるか判定する。

**無影響を確認できる場合**

- 自動実行
- 再測定
- 外観・動作比較
- 問題があればロールバック
- 問題なければ採用

**無影響を確定できない場合**

- 改善見込みと影響をユーザーへ提示
- 許可された場合のみ実施
- 再測定・比較

**明確な機能変更の場合**

- ユーザーの事前許可を必須とする

その後、目標Performanceに到達するか、追加の無影響最適化がなくなるまで繰り返す。

最後に総合検証を実施し、作業シーンを保存する。

---

# 29. 実測知識の扱い

過去の最適化で得られた以下の結果は参考例として保持する。

- 4Kから2KへのTexture制限で約164MBから76MBへ削減
- MeshiaのShoes 0.55は成立したが0.45ではArtifact発生
- 異なるPhysBone Chain長のMergeで約1.5cmの差が発生した例
- Endpoint長の外れ値を平均化すると数cm規模のMotion差が生じた例
- Merge Boneによって揺れ幅が大幅に減少した例
- 同じShaderのMaterialでも多数のProperty差が存在した例
- 隠れGeometry削除によって膝・脛付近に穴が生じた例
- PrefabSafeSetのRemoveでSet全体が消えた例

これらは固定設定や普遍的な閾値ではない。

新しいアバターでは必ず再測定・再検証する。

---

# 30. 知識ベースの更新ルール

新しいアバターで軽量化した後、知見を次の2種類に分ける。

## 一般知識

別のアバターでも再利用できる条件・注意点・検証方法。

本書へ追記する。

## アバター固有知識

- 特定Rendererの削減率
- 特定PhysBoneのEndpoint値
- 特定衣装のMaterial構成
- 特定Menuの不要機能
- 特定アバターの最適化結果

個別作業ログへ記録する。

同じ現象が複数のアバターで確認された場合、一般知識として整理し直す。

---

# 31. 最終原則

VRChatアバターの軽量化で重要なのは、単純にTrianglesやTextureを減らすことではない。

**最終ビルド状態を測定し、現在のボトルネックに対して、最も副作用の小さい処理から順に適用すること。**

Claude Codeは、安全性を検証できる非破壊最適化については自律的に実行してよい。

特に、

- 不要ObjectのEditorOnly化
- AAOによる隠れポリゴンの非破壊削除
- Texture Size最適化
- Texture Atlas
- Renderer / Material統合
- 安全なPhysBone統合
- 段階的なMesh Simplification

を積極的に検討する。

一方、機能や外観が変わる可能性を排除できない場合は、ユーザーへ判断を求める。

すべての最適化は、

**測定 → 最適化 → 再測定 → 比較 → 採用またはロールバック**

を基本とする。

「軽くなったはず」ではなく、Actual Performanceと外観・動作比較によって軽量化の成果を確認する。

---

# 参考資料

- [VRChat Avatar Performance Ranking System](https://creators.vrchat.com/avatars/avatar-performance-ranking-system/)
- [VRChat Avatar Optimization Tips](https://creators.vrchat.com/avatars/avatar-optimizing-tips/)
- [Avatar Optimizer Documentation](https://vpm.anatawa12.com/avatar-optimizer/en/)
- [AAO Trace and Optimize](https://vpm.anatawa12.com/avatar-optimizer/en/docs/reference/trace-and-optimize/)
- [AAO Merge Skinned Mesh](https://vpm.anatawa12.com/avatar-optimizer/en/docs/reference/merge-skinned-mesh/)
- [AAO Merge Material](https://vpm.anatawa12.com/avatar-optimizer/en/docs/reference/merge-material/)
- [AAO Max Texture Size](https://vpm.anatawa12.com/avatar-optimizer/en/docs/reference/max-texture-size/)
- [TexTransTool Documentation](https://ttt.rs64.net/)
- [NDMF Documentation](https://ndmf.nadena.dev/)
- [Actual Performance Window](https://vpm.anatawa12.com/gists/ja/docs/reference/actual-performance-window/)

各ツールの具体的な挙動は導入バージョンによって変わるため、実作業時には対応する公式ドキュメントとBuild Logを確認する。
