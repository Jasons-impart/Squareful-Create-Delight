# Quark 3D 书架技术路线（Technical Notes）

本扩展为 Quark 各木材书架（`quark:*_bookshelf`）提供与方纹原版 `minecraft:bookshelf` 一致的 3D 随机书架外观：书本层完全复用方纹，仅框架（side/top）按木材替换。

## 1. 覆盖范围

10 种木材（`config/quark-common.toml` 中 Ancient Wood / Azalea Wood / Blossom Trees 已禁用，故不覆盖）：

`acacia, bamboo, birch, cherry, crimson, dark_oak, jungle, mangrove, spruce, warped`

## 2. 文件结构（每木材 19 个，共 190 个）

```
assets/quark/blockstates/{w}_bookshelf.json      # 16 条加权随机 variant
assets/quark/models/block/{w}_bookshelf_3d*.json  # 16 个（4 布局 × 4 旋转）
assets/quark/textures/block/{w}_bookshelf3d_side.png
assets/quark/textures/block/{w}_bookshelf3d_top.png
```

不覆盖 `models/item/{w}_bookshelf.json` —— 物品图标保持扁平（与方纹原版策略一致）。

## 3. 模型构造（父链复用）

每个木材模型是 8 行的小 JSON，**parent 指向方纹的 minecraft 3D 模型**，只覆盖 3 个纹理变量，书本纹理（b1/b2/b3）自动继承：

```json
{
    "parent": "minecraft:block/bookshelf_3d",
    "textures": {
        "particle": "quark:block/spruce_bookshelf3d_side",
        "side": "quark:block/spruce_bookshelf3d_side",
        "top": "quark:block/spruce_bookshelf3d_top"
    }
}
```

blockstate 照抄方纹 `assets/minecraft/blockstates/bookshelf.json` 的权重结构（3/3/3/3, 1/1/1/1, 2/2/2/2, 1/1/1/1），模型名换成 `quark:block/{w}_bookshelf_3d{,_r,_rr,_rrr,_2...}`。

依赖：方纹主包（`minecraft:block/bookshelf_3d*` 与 `minecraft:block/bookshelf3d_book*`），符合本扩展"非独立扩展"声明。

## 4. 框架纹理生成：调色板交换（Palette Swap）

### 4.1 方纹色板结构（关键发现）

- 每木材所有木纹基于 **9 色调色板**：`{w}_planks.png`（7 色）+ `{w}_planks4/_end/_end1`（补 2 深色）
- `bookshelf3d_top.png` = 纯色板（9 色全部属于 planks 色板）
- `bookshelf3d_side.png` = 5 个色板色 + **4 个手绘深色阴影**（不在色板内）：
  `56,43,32 / 90,69,52 / 81,62,47 / 72,55,42`（相对 oak 最深色 122,95,56 的亮度比 0.463 / 0.744 / 0.669 / 0.594）
- 扁平 `bookshelf.png` = planks 色 + 手绘书（木质区域逐像素等于 planks）
- bamboo 例外：11 色、无 planks4（走 rank 比例映射）

### 4.2 映射算法

1. 提取 oak 参考色板（`oak_planks` + `oak_planks4/_end/_end1` 并集），按亮度升序
2. 提取目标木材色板（`{w}_planks*` 并集），按亮度升序
3. 建 LUT：oak 第 i 色 → 木材色板第 `round(i·(m-1)/(n-1))` 色（rank 比例映射，兼容不等长色板）
4. `top`：全图 LUT 映射
5. `side`：色板色走 LUT；4 个深色阴影 = 木材最深色 × 对应亮度比（逐通道，截断 255）

结果：**100% 复刻方纹纹理结构（描边/木纹排列），仅换色板**，与游戏内方纹木板颜色一致。

### 4.3 已验证

- 所有 top 纹理像素全部命中各自木材色板
- 所有 side 纹理 = 5 色板色 + 4 个正确缩放的深色阴影
- LUT 逐像素核对无误（如 oak 157,125,74 → spruce 107,76,47，亮度秩精确对应）

## 5. 生成脚本

| 脚本 | 作用 |
|------|------|
| `scripts/gen-quark-bookshelf-3d.ps1` | 生成 16 模型 + 1 blockstate（`-Woods a,b,c`） |
| `scripts/gen-bookshelf-frame-texture.ps1` | 调色板交换生成 side/top（`-Woods a,b,c`） |

注意事项：
- 脚本须为 **UTF-8 带 BOM**（PS 5.1 按 ANSI 读取，含中文路径会乱码）
- 生成的 JSON 必须**无 BOM**（Gson 解析可能报错）
- 纹理全部来自方纹包（`resourcepacks/Squareful方纹v26.3.0/`），不依赖原版 jar

## 6. 发布提醒

改动生效需重新打包 `packwiz-files/resourcepacks/Squareful-Create_Delight_Remake.zip` 并更新 `resourcepacks/squareful-create-delight-remake.pw.toml` 的 hash（走 packwiz-assets 流程）。本地开发直接用 `resourcepacks/Squareful_CDR_pack` 文件夹 + F3+T 重载即可。