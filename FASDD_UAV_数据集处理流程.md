# FASDD_UAV 数据集处理流程（可复现版）

本文档记录从下载 FASDD_UAV 到生成去重、筛选后的干净数据集 `FASDD_UAV_clean` 的完整流程。按顺序执行每一步，并对照每一步的"预期输出"，即可得到与原作者完全一致的数据集。

- 运行环境：AutoDL 实例（Linux），Python ≥ 3.10（`Path.hardlink_to` 需要 3.10+）
- 数据来源：Wang et al., An open flame and smoke detection dataset for deep learning in remote sensing based fire detection, Geo-spatial Information Science, 28(2): 511–526, 2025. DOI: 10.1080/10095020.2024.2347922
- 下载页面：Science Data Bank，DOI 10.57760/sciencedb.j00104.00103
- 最终产物：`/root/autodl-tmp/FASDD_UAV_clean`
- 类别编号：`0 = fire, 1 = smoke`（与 D-Fire 的 `0 = smoke` 相反，注意不要混用）

---

## 0. 准备工作

### 0.1 目录约定

| 路径 | 内容 |
|---|---|
| `/root/autodl-tmp/FASDD_UAV.zip` | 下载的压缩包 |
| `/root/autodl-tmp/FASDD_UAV_raw/` | 解压后的原始数据 |
| `/root/autodl-tmp/FASDD_UAV_yolo/` | 按官方划分整理的 YOLO 结构（硬链接） |
| `/root/autodl-tmp/fasdd_check/` | 中间统计文件（哈希、图像属性等） |
| `/root/autodl-tmp/FASDD_UAV_clean/` | 最终干净数据集（硬链接） |
| `/root/autodl-tmp/fasdd_scripts/` | 本文档中的所有脚本 |

```bash
mkdir -p /root/autodl-tmp/fasdd_scripts /root/autodl-tmp/fasdd_check
```

以下所有脚本都保存到 `/root/autodl-tmp/fasdd_scripts/`。脚本内部全部使用绝对路径，从哪个目录运行都可以。

### 0.2 依赖与版本

```bash
pip install imagehash==4.3.1        # 会一并安装 scipy、PyWavelets
pip show imagehash pillow numpy | grep -E "Name|Version"
```

pHash 的结果依赖 Pillow 和 imagehash 的版本。复现时请记录上面命令输出的版本号。如果第 5 步的数字和预期不一致，先检查版本。

### 0.3 磁盘空间

```bash
df -h /root/autodl-tmp
```

压缩包 13.87 GB，解压后体积相近，建议可用空间 ≥ 30 GB。后续的整理和构建都用硬链接，不额外占用空间。

下载、解压和第 3 到第 9 步只需要 CPU，在 AutoDL 无卡模式下执行即可。数据在 `/root/autodl-tmp` 中，切换到 GPU 模式后仍然保留。

---

## 1. 下载 FASDD_UAV.zip

ScienceDB 的下载地址是点击下载按钮后临时生成的，需要先在本地浏览器获取链接，再交给服务器下载。文件不会经过本地电脑。

### 1.1 在本地浏览器获取下载链接

1. 用 Chrome 打开数据集页面：https://www.scidb.cn/en/detail?dataSetId=ce9c9400b44148e1b0a749f5c3eb0bda （如果要求登录，先登录）
2. 在文件列表中找到 `FASDD_UAV.zip`（13.87 GB，MD5 `c5ea9651ca672fc128c6f17d0797c2f2`），点击下载图标。只下载这个文件，不需要 `FASDD.zip` 全集，也不需要 RS 相关的压缩包。
3. 下载开始后按 `Ctrl + J` 打开下载记录，右键 `FASDD_UAV.zip` 这一条，选择"复制链接地址"。
4. 取消本地下载。

链接形如 `https://china.scidb.cn/download?fileId=...&traceId=...`，是临时链接，复制后要尽快使用。

如果用这个链接在服务器上下载到的是 HTML 网页（见 1.3 的检查），改用这种方式：按 F12 打开 Network 面板，再点击下载，在对应请求上右键，选择 Copy，再选 Copy as cURL (bash)，把整条命令粘贴到服务器执行，并在末尾加上 `-L -C - --retry 5 -o FASDD_UAV.zip`。这条命令里包含你的登录 Cookie，不要分享给别人。

### 1.2 在服务器上下载

```bash
tmux new -s fasdd
cd /root/autodl-tmp
curl -L -C - --retry 5 -o FASDD_UAV.zip '把复制的链接粘贴在这里'
# Ctrl+B 再按 D 挂到后台；用 tmux attach -t fasdd 回来查看进度
```

- 链接必须用单引号括起来，因为里面有 `&`、`?`。
- `-C -` 支持断点续传。中断后重新执行同一条命令即可。如果报 403（链接过期），回浏览器重新复制链接再执行。
- 参考速度约 5.5 MB/s，全程约 40 分钟。下载速度与 CPU/GPU 无关。

### 1.3 校验

```bash
cd /root/autodl-tmp
ls -lh FASDD_UAV.zip          # 约 13.9 GB
file FASDD_UAV.zip            # 应显示 Zip archive data
echo "c5ea9651ca672fc128c6f17d0797c2f2  FASDD_UAV.zip" | md5sum -c -
```

预期输出：`FASDD_UAV.zip: OK`

---

## 2. 选择性解压

压缩包根目录是 `annotations/` 和 `images/`。`annotations/` 下有 `YOLO_UAV`、`COCO_UAV` 等子目录。只解压图片、YOLO 标注和 COCO json（json 只用来确认类别名称）。

```bash
cd /root/autodl-tmp
unzip -l FASDD_UAV.zip | awk '{print $4}' | cut -d/ -f1-3 | sort | uniq -c | sort -rn | head -20   # 先确认结构

tmux new -s unz
mkdir -p FASDD_UAV_raw
unzip -q FASDD_UAV.zip 'images/*' 'annotations/YOLO_UAV/*' -d FASDD_UAV_raw
unzip -q FASDD_UAV.zip 'annotations/*COCO*/*.json' -d FASDD_UAV_raw 2>/dev/null; echo "COCO json 解压退出码: $?"
```

如果中途中断，重新执行 `unzip -o FASDD_UAV.zip 'images/*' 'annotations/YOLO_UAV/*' -d FASDD_UAV_raw`（`-o` 表示覆盖写了一半的文件）。

### 2.1 检查解压结果

```bash
cd /root/autodl-tmp/FASDD_UAV_raw
Y=./annotations/YOLO_UAV
wc -l $Y/train.txt $Y/val.txt $Y/test.txt
head -3 $Y/train.txt
ls $Y/labels | wc -l
find . -type f \( -iname "*.jpg" -o -iname "*.png" \) | wc -l
find . -name "*.json"
```

预期输出：
- `wc -l`：train 12550、val 8364、test 4180（文件末尾没有换行，所以实际条目数分别是 12551、8365、4181）
- `train.txt` 每行形如 `./images/bothFireAndSmoke_UAV004735.jpg`
- 标签文件 25097 个，图片 25097 张
- COCO json：`./annotations/COCO_UAV/Annotations/{train,val,test}.json`

---

## 3. 整理成 YOLO 结构（按官方划分）

保存为 `/root/autodl-tmp/fasdd_scripts/organize_fasdd_uav.py`：

```python
# 将 FASDD_UAV 整理为 Ultralytics 标准结构（硬链接，不占额外空间，不改动原始解压文件）
# 输出: FASDD_UAV_yolo/{images,labels}/{train,val,test}
import json, os, re
from pathlib import Path
from collections import Counter

RAW = Path("/root/autodl-tmp/FASDD_UAV_raw")
OUT = Path("/root/autodl-tmp/FASDD_UAV_yolo")
IMG_EXT = {".jpg", ".jpeg", ".png", ".bmp"}

if OUT.exists():
    raise SystemExit(f"{OUT} 已存在，确认后手动删除或换名，避免覆盖")

yolo_dir = next(RAW.rglob("YOLO_UAV"))
label_dir = yolo_dir / "labels"

# 建立文件名 -> 路径索引（txt 里路径格式不确定，统一按文件名匹配）
img_index = {}
for p in RAW.rglob("*"):
    if p.suffix.lower() in IMG_EXT and "annotations" not in p.parts:
        if p.name in img_index:
            raise SystemExit(f"图片文件名重复: {p.name}，需要改为按相对路径匹配")
        img_index[p.name] = p
lbl_index = {p.stem: p for p in label_dir.rglob("*.txt")}
print(f"图片 {len(img_index)} 张，标签文件 {len(lbl_index)} 个")

def read_split(name):
    names = []
    for ln in (yolo_dir / f"{name}.txt").read_text(encoding="utf-8", errors="ignore").splitlines():
        ln = ln.strip()
        if ln:
            names.append(re.split(r"[\\/]", ln)[-1])   # 兼容 Windows / Linux 路径
    return names

splits = {s: read_split(s) for s in ("train", "val", "test")}

# 划分之间是否重叠
sets = {s: set(v) for s, v in splits.items()}
for a, b in (("train", "val"), ("train", "test"), ("val", "test")):
    inter = sets[a] & sets[b]
    print(f"{a}∩{b}: {len(inter)} 张（同名重叠，应为 0）")

cls_count, report = Counter(), {}
for s, names in splits.items():
    (OUT / "images" / s).mkdir(parents=True)
    (OUT / "labels" / s).mkdir(parents=True)
    miss_img = miss_lbl = empty = bad = 0
    for n in names:
        img = img_index.get(n)
        if img is None:
            miss_img += 1
            continue
        os.link(img, OUT / "images" / s / img.name)
        lbl = lbl_index.get(Path(n).stem)
        dst = OUT / "labels" / s / (Path(n).stem + ".txt")
        if lbl is None:
            dst.write_text("")             # 无标签文件按负样本处理
            miss_lbl += 1
            continue
        lines = [l for l in lbl.read_text().splitlines() if l.strip()]
        if not lines:
            empty += 1
        for l in lines:
            parts = l.split()
            if len(parts) != 5:
                bad += 1
            else:
                cls_count[(s, int(float(parts[0])))] += 1
        os.link(lbl, dst)
    report[s] = dict(listed=len(names), missing_image=miss_img, missing_label=miss_lbl,
                     empty_label=empty, bad_lines=bad)

for s, r in report.items():
    print(s, r)
print("各划分类别计数 (split, class_id): count")
for k in sorted(cls_count):
    print("  ", k, cls_count[k])

# 从 COCO json 读取类别定义（如果有）
for j in RAW.rglob("*.json"):
    try:
        cats = json.loads(j.read_text())["categories"]
        print(f"COCO 类别定义 ({j.name}):", [(c['id'], c['name']) for c in cats])
        break
    except Exception:
        continue
```

```bash
python /root/autodl-tmp/fasdd_scripts/organize_fasdd_uav.py
```

预期输出：

```
图片 25097 张，标签文件 25097 个
train∩val: 0 / train∩test: 0 / val∩test: 0
train {'listed': 12551, 'missing_image': 0, 'missing_label': 0, 'empty_label': 5994, 'bad_lines': 0}
val   {'listed': 8365,  'missing_image': 0, 'missing_label': 0, 'empty_label': 3995, 'bad_lines': 0}
test  {'listed': 4181,  'missing_image': 0, 'missing_label': 0, 'empty_label': 1997, 'bad_lines': 0}
('test', 0) 6049   ('test', 1) 2877
('train', 0) 18169 ('train', 1) 8607
('val', 0) 12090   ('val', 1) 5738
COCO 类别定义 (test.json): [(0, 'fire'), (1, 'smoke')]
```

在 `FASDD_UAV_yolo` 下写入类别配置（这个 yaml 只用于官方划分，最终实验不用它）：

```bash
cat > /root/autodl-tmp/FASDD_UAV_yolo/data.yaml << 'EOF'
path: /root/autodl-tmp/FASDD_UAV_yolo
train: images/train
val: images/val
test: images/test
names:
  0: fire
  1: smoke
EOF
```

---

## 4. 核对类别编号与前缀分布

保存为 `/root/autodl-tmp/fasdd_scripts/check_classes.py`：

```python
# 统计文件名前缀 × 划分，并为 class 0 / class 1 各画 12 张样例拼图
import random, re
from pathlib import Path
from collections import Counter
from PIL import Image, ImageDraw

ROOT = Path("/root/autodl-tmp/FASDD_UAV_yolo")
OUT = Path("/root/autodl-tmp/fasdd_check"); OUT.mkdir(exist_ok=True)
random.seed(0)

# 1) 文件名前缀分布（前缀 = 下划线前的部分）
for s in ("train", "val", "test"):
    c = Counter(re.split(r"_", p.stem)[0] for p in (ROOT / "images" / s).iterdir())
    print(s, dict(c))

# 2) 每个类别抽 12 张含该类的图，只画该类的框（红色），拼成 4x3 网格
lbl_dir = ROOT / "labels" / "train"
by_cls = {0: [], 1: []}
for f in lbl_dir.glob("*.txt"):
    cls = {int(l.split()[0]) for l in f.read_text().splitlines() if l.strip()}
    for c in cls & {0, 1}:
        by_cls[c].append(f)

for c, files in by_cls.items():
    tiles = []
    for f in random.sample(files, min(12, len(files))):
        img = Image.open(ROOT / "images" / "train" / (f.stem + ".jpg")).convert("RGB")
        W, H = img.size; d = ImageDraw.Draw(img)
        for l in f.read_text().splitlines():
            k, x, y, w, h = l.split()
            if int(k) != c: continue
            x, y, w, h = float(x)*W, float(y)*H, float(w)*W, float(h)*H
            d.rectangle([x-w/2, y-h/2, x+w/2, y+h/2], outline=(255, 0, 0), width=max(2, W//200))
        tiles.append(img.resize((480, 360)))
    grid = Image.new("RGB", (480*4, 360*3))
    for i, t in enumerate(tiles):
        grid.paste(t, ((i % 4)*480, (i // 4)*360))
    grid.save(OUT / f"class{c}_samples.jpg", quality=85)
    print(f"class {c}: {len(files)} 张图含该类，样例已保存 {OUT}/class{c}_samples.jpg")
```

```bash
python /root/autodl-tmp/fasdd_scripts/check_classes.py
```

预期输出：

```
train {'bothFireAndSmoke': 3911, 'fire': 105, 'neitherFireNorSmoke': 5994, 'smoke': 2541}
val   {'bothFireAndSmoke': 2607, 'fire': 70,  'neitherFireNorSmoke': 3995, 'smoke': 1693}
test  {'bothFireAndSmoke': 1303, 'fire': 35,  'neitherFireNorSmoke': 1997, 'smoke': 846}
class 0: 4016 张图含该类
class 1: 6450 张图含该类
```

人工查看 `fasdd_check/class0_samples.jpg`（应该框的是火）和 `class1_samples.jpg`（应该框的是烟），确认 `0=fire, 1=smoke`。

拼图里也能看到这个数据集的特点：火框粒度很不统一（有时拆成很多小框，有时整图一个大框）；满屏是烟的图，烟框就是整张图；还混有固定瞭望塔相机拍的远处淡烟。

---

## 5. pHash 近重复分析

这一步生成 `phash.npz`，第 9 步需要用它。保存为 `/root/autodl-tmp/fasdd_scripts/dedup_analysis.py`：

```python
# 计算 pHash，统计跨划分近重复比例，以及不同阈值下的簇数
import numpy as np, imagehash
from pathlib import Path
from PIL import Image

ROOT = Path("/root/autodl-tmp/FASDD_UAV_yolo")
CACHE = Path("/root/autodl-tmp/fasdd_check/phash.npz")
CACHE.parent.mkdir(parents=True, exist_ok=True)
SPLITS = ("train", "val", "test")

if CACHE.exists():
    z = np.load(CACHE, allow_pickle=True); names, splits, H = z["names"], z["splits"], z["H"]
else:
    names, splits, H = [], [], []
    for s in SPLITS:
        for i, p in enumerate(sorted((ROOT / "images" / s).iterdir())):
            with Image.open(p) as im:
                h = imagehash.phash(im).hash.flatten()
            H.append(np.packbits(h).view(np.uint64)[0]); names.append(p.name); splits.append(s)
            if i % 2000 == 0: print(s, i)
    names, splits, H = np.array(names), np.array(splits), np.array(H, dtype=np.uint64)
    np.savez(CACHE, names=names, splits=splits, H=H)

POP = np.array([bin(i).count("1") for i in range(256)], dtype=np.uint8)
def ham(block, allh):                       # block:(b,) allh:(n,) -> (b,n) 汉明距离
    x = (block[:, None] ^ allh[None, :]).view(np.uint8).reshape(len(block), len(allh), 8)
    return POP[x].sum(-1, dtype=np.uint16)

n = len(H); THRS = (4, 8, 12)
min_to_train = np.full(n, 64, dtype=np.uint16)
parent = {t: np.arange(n) for t in THRS}
def find(p, i):
    while p[i] != i: p[i] = p[p[i]]; i = p[i]
    return i

tr = np.where(splits == "train")[0]
for st in range(0, n, 500):
    idx = np.arange(st, min(st + 500, n))
    D = ham(H[idx], H)
    min_to_train[idx] = np.where(splits[idx] == "train", 64,
                                 D[:, tr].min(1))           # 非 train 图到 train 的最近距离
    for t in THRS:
        ii, jj = np.nonzero(D <= t)
        for a, b in zip(idx[ii], jj):
            if a < b:
                ra, rb = find(parent[t], a), find(parent[t], b)
                if ra != rb: parent[t][ra] = rb
    if st % 5000 == 0: print("pairs", st)

for s in ("val", "test"):
    m = min_to_train[splits == s]
    print(s, {f"<= {t}": f"{(m <= t).mean():.1%}" for t in THRS}, "（与 train 近重复的比例）")
for t in THRS:
    roots = np.array([find(parent[t], i) for i in range(n)])
    sizes = np.bincount(np.unique(roots, return_inverse=True)[1])
    print(f"阈值 {t}: 簇数 {len(sizes)}（即去重后剩余图像数），最大簇 {sizes.max()}，单图簇 {(sizes == 1).sum()}")
```

```bash
tmux new -s dedup
python /root/autodl-tmp/fasdd_scripts/dedup_analysis.py
```

逐张计算哈希大约需要十几到几十分钟（无卡模式更慢）。预期输出：

```
val  {'<= 4': '91.2%', '<= 8': '95.6%', '<= 12': '97.2%'}（与 train 近重复的比例）
test {'<= 4': '90.7%', '<= 8': '95.5%', '<= 12': '97.2%'}（与 train 近重复的比例）
阈值 4:  簇数 2296，最大簇 1013，单图簇 1495
阈值 8:  簇数 1159，最大簇 6046，单图簇 780
阈值 12: 簇数 694， 最大簇 14972，单图簇 479
```

结论：官方 test 中约 91% 的图与 train 近重复（与 Drones 2026 年 Cross-Dataset Evaluation of YOLOv8 for UAV Fire and Smoke Detection 一文报告的 90.7% 一致），所以官方划分不能用于评估泛化能力，需要重新划分。

---

## 6. 统计图像属性（用于识别瞭望塔相机图）

保存为 `/root/autodl-tmp/fasdd_scripts/filter_stats.py`：

```python
# 统计每张图的客观属性，评估各条筛选规则的影响（只读，不改文件）
import numpy as np, csv
from pathlib import Path
from PIL import Image

ROOT = Path("/root/autodl-tmp/FASDD_UAV_yolo")
OUT = Path("/root/autodl-tmp/fasdd_check/image_props.csv")
OUT.parent.mkdir(parents=True, exist_ok=True)

rows = []
for s in ("train", "val", "test"):
    for img_p in sorted((ROOT / "images" / s).iterdir()):
        lbl = ROOT / "labels" / s / (img_p.stem + ".txt")
        with Image.open(img_p) as im:
            W, H = im.size
            g = np.asarray(im.convert("L").resize((64, 64)), dtype=np.float32)
        corner = np.mean([g[:6, :6].mean(), g[:6, -6:].mean(), g[-6:, :6].mean(), g[-6:, -6:].mean()])
        center = g[24:40, 24:40].mean()
        a = np.loadtxt(lbl, ndmin=2) if lbl.stat().st_size else np.zeros((0, 5))
        area = a[:, 3] * a[:, 4] if len(a) else np.array([])
        min_side_px = np.minimum(a[:, 3] * W, a[:, 4] * H) if len(a) else np.array([])
        rows.append(dict(split=s, image=img_p.name, prefix=img_p.stem.split("_")[0], W=W, H=H,
                         n_fire=int((a[:, 0] == 0).sum()) if len(a) else 0,
                         n_smoke=int((a[:, 0] == 1).sum()) if len(a) else 0,
                         max_box_area=float(area.max()) if len(area) else 0.0,
                         max_box_side_px=float(min_side_px.max()) if len(min_side_px) else 0.0,
                         fisheye_like=int(corner < 20 and center > 40)))

with open(OUT, "w", newline="") as f:
    w = csv.DictWriter(f, fieldnames=rows[0].keys()); w.writeheader(); w.writerows(rows)

pos = [r for r in rows if r["n_fire"] + r["n_smoke"] > 0]
print(f"正样本图 {len(pos)} / 总 {len(rows)}")
res = np.array([(r["W"], r["H"]) for r in rows])
print(f"分辨率中位数 {np.median(res[:,0]):.0f}x{np.median(res[:,1]):.0f}，短边 P5={np.percentile(res.min(1),5):.0f}")
print(f"疑似鱼眼/瞭望塔图: {sum(r['fisheye_like'] for r in rows)}")
for t in (0.8, 0.9, 0.95):
    print(f"含面积 > {t:.0%} 大框的正样本图: {sum(r['max_box_area'] > t for r in pos)}")
for px in (8, 16, 32):
    print(f"全部框短边 < {px}px 的正样本图（最大框也很小）: {sum(r['max_box_side_px'] < px for r in pos)}")
```

为了严格复现，这里不要加 `im.draft(...)` 加速。draft 会改变缩略图的像素值，`fisheye_like` 的判断结果可能因此出现细微差异。

```bash
tmux new -s stats
python /root/autodl-tmp/fasdd_scripts/filter_stats.py
```

运行时没有进度输出，需要十几到几十分钟。预期输出：

```
正样本图 13111 / 总 25097
分辨率中位数 2048x1536，短边 P5=720
疑似鱼眼/瞭望塔图: 1118
含面积 > 80% 大框的正样本图: 2972
含面积 > 90% 大框的正样本图: 1737
含面积 > 95% 大框的正样本图: 1205
全部框短边 < 8px 的正样本图（最大框也很小）: 0
全部框短边 < 16px 的正样本图（最大框也很小）: 0
全部框短边 < 32px 的正样本图（最大框也很小）: 46
```

大框统计只作参考，最终决定保留大框，不按面积筛选。

---

## 7. 统计 640 尺度下的极小框

YOLO 训练时会把图缩放到 640，所以要按缩放后的尺寸判断目标是否过小。保存为 `/root/autodl-tmp/fasdd_scripts/tiny_box_stats.py`：

```python
# 按 YOLO 输入 640 尺度统计框的短边，评估“含极小框即删整图”会损失多少（只读）
import csv
import numpy as np
from pathlib import Path
from PIL import Image

ROOT = Path("/root/autodl-tmp/FASDD_UAV_yolo")
CHECK = Path("/root/autodl-tmp/fasdd_check")
IMGSZ = 640
THRS = (4, 6, 8, 12, 16)           # 640 尺度下的短边像素阈值
NAMES = {0: "fire", 1: "smoke"}

sides = {0: [], 1: []}
img_rows = []
for s in ("train", "val", "test"):
    for lbl in sorted((ROOT / "labels" / s).glob("*.txt")):
        if not lbl.stat().st_size:
            img_rows.append((s, lbl.stem, np.inf)); continue
        with Image.open(ROOT / "images" / s / (lbl.stem + ".jpg")) as im:
            W, H = im.size
        a = np.loadtxt(lbl, ndmin=2)
        scale = IMGSZ / max(W, H)                         # letterbox 缩放比例
        short = np.minimum(a[:, 3] * W, a[:, 4] * H) * scale
        for c in NAMES:
            sides[c].extend(short[a[:, 0] == c].tolist())
        img_rows.append((s, lbl.stem, float(short.min())))

for c, v in sides.items():
    v = np.array(v)
    print(f"{NAMES[c]:>5} 框 {len(v)} 个，640 尺度短边 P5/P25/P50 = "
          + "/".join(f"{np.percentile(v, q):.1f}" for q in (5, 25, 50)) + " px；"
          + "，".join(f"<{t}px: {(v < t).mean():.1%}" for t in THRS))

mins = np.array([r[2] for r in img_rows])
pos = np.isfinite(mins)
print(f"\n正样本图 {pos.sum()} 张")
for t in THRS:
    print(f"含 <{t}px 框的正样本图: {(mins[pos] < t).sum()} 张（{(mins[pos] < t).mean():.1%}），删除后剩 {(mins[pos] >= t).sum()} 张")

with open(CHECK / "min_box_side640.csv", "w", newline="") as f:
    w = csv.writer(f); w.writerow(["split", "stem", "min_side_640"])
    w.writerows(img_rows)
print(f"\n逐图最小框短边已写入 {CHECK / 'min_box_side640.csv'}")
```

```bash
python /root/autodl-tmp/fasdd_scripts/tiny_box_stats.py
```

只读取图片尺寸，1 到 3 分钟即可完成。预期输出：

```
 fire 框 36308 个，640 尺度短边 P5/P25/P50 = 6.0/10.0/17.5 px；<4px: 0.6%，<6px: 3.8%，<8px: 12.8%，<12px: 32.0%，<16px: 45.9%
smoke 框 17222 个，640 尺度短边 P5/P25/P50 = 18.4/48.8/195.5 px；<4px: 0.0%，<6px: 0.0%，<8px: 0.1%，<12px: 0.8%，<16px: 3.0%

正样本图 13111 张
含 <4px 框的正样本图: 138 张（1.1%），删除后剩 12973 张
含 <6px 框的正样本图: 819 张（6.2%），删除后剩 12292 张
含 <8px 框的正样本图: 2211 张（16.9%），删除后剩 10900 张
含 <12px 框的正样本图: 4291 张（32.7%），删除后剩 8820 张
含 <16px 框的正样本图: 5558 张（42.4%），删除后剩 7553 张
```

阈值按预设规则确定：从 8 px 开始，如果删除的正样本超过 30% 就降到 6 px。8 px 删除 16.9%，所以 `MIN_SIDE_640 = 8`。

---

## 8. 识别固定瞭望塔相机图

### 8.1 检查初步判断（可选）

第 6 步的 `fisheye_like`（四角暗、中心亮）会把夜间火场近景误判进来。保存为 `/root/autodl-tmp/fasdd_scripts/check_fisheye.py`：

```python
# 随机抽 24 张被判为鱼眼的图拼成网格，人工确认有没有把夜景火场误判进来
import csv, random
from pathlib import Path
from PIL import Image

ROOT = Path("/root/autodl-tmp/FASDD_UAV_yolo")
CHECK = Path("/root/autodl-tmp/fasdd_check")
random.seed(0)
rows = [r for r in csv.DictReader(open(CHECK / "image_props.csv")) if r["fisheye_like"] == "1"]
pick = random.sample(rows, min(24, len(rows)))
grid = Image.new("RGB", (320 * 6, 240 * 4))
for i, r in enumerate(pick):
    with Image.open(ROOT / "images" / r["split"] / r["image"]) as im:
        im.draft("RGB", (320, 240))
        grid.paste(im.convert("RGB").resize((320, 240)), ((i % 6) * 320, (i // 6) * 240))
grid.save(CHECK / "fisheye_samples.jpg", quality=85)
print("前缀分布:", {p: sum(r["prefix"] == p for r in rows) for p in {r["prefix"] for r in rows}})
```

```bash
python /root/autodl-tmp/fasdd_scripts/check_fisheye.py
```

预期输出：`前缀分布: {'bothFireAndSmoke': 304, 'smoke': 79, 'fire': 131, 'neitherFireNorSmoke': 604}`（字典顺序可能不同）。拼图中会同时出现白天鱼眼瞭望塔图、夜间地平线瞭望塔图，以及被误判的夜间火场近景。

### 8.2 用火焰色占比救回夜间火场近景

保存为 `/root/autodl-tmp/fasdd_scripts/check_tower_cam.py`：

```python
# 在原“鱼眼”候选中用火焰色占比救回夜间火场近景，生成 image_props_v2.csv
import csv, random
import numpy as np
from pathlib import Path
from collections import Counter
from PIL import Image

ROOT  = Path("/root/autodl-tmp/FASDD_UAV_yolo")
CHECK = Path("/root/autodl-tmp/fasdd_check")
FIRE_FRAC_T = 0.02          # 亮橙色像素占比阈值
random.seed(0)

def fire_frac(path):
    with Image.open(path) as im:
        im.draft("RGB", (256, 192))
        hsv = np.asarray(im.convert("RGB").resize((128, 96)).convert("HSV"))
    h, s, v = hsv[..., 0].astype(int), hsv[..., 1], hsv[..., 2]
    # PIL 的 HSV 色相范围 0-255：0-42 约对应红/橙/黄（0-60°）
    return float((((h <= 42) | (h >= 245)) & (s >= 100) & (v >= 180)).mean())

rows = list(csv.DictReader(open(CHECK / "image_props.csv")))
tower, rescued = [], []
for r in rows:
    r["fire_frac"], r["tower_cam"] = "", "0"
    if r["fisheye_like"] == "1":
        ff = fire_frac(ROOT / "images" / r["split"] / r["image"])
        r["fire_frac"] = f"{ff:.4f}"
        if ff < FIRE_FRAC_T:
            r["tower_cam"] = "1"; tower.append(r)
        else:
            rescued.append(r)

with open(CHECK / "image_props_v2.csv", "w", newline="") as f:
    w = csv.DictWriter(f, fieldnames=rows[0].keys()); w.writeheader(); w.writerows(rows)

def grid(items, out):
    pick = random.sample(items, min(24, len(items)))
    g = Image.new("RGB", (320 * 6, 240 * 4))
    for i, r in enumerate(pick):
        with Image.open(ROOT / "images" / r["split"] / r["image"]) as im:
            im.draft("RGB", (320, 240))
            g.paste(im.convert("RGB").resize((320, 240)), ((i % 6) * 320, (i // 6) * 240))
    g.save(out, quality=85)

grid(tower, CHECK / "tower_cam_samples.jpg")
grid(rescued, CHECK / "rescued_samples.jpg")
print(f"判为瞭望塔（将删除）: {len(tower)}，前缀 {dict(Counter(r['prefix'] for r in tower))}")
print(f"救回（火场近景）: {len(rescued)}，前缀 {dict(Counter(r['prefix'] for r in rescued))}")
```

```bash
python /root/autodl-tmp/fasdd_scripts/check_tower_cam.py
```

预期输出：

```
判为瞭望塔（将删除）: 785，前缀 {'bothFireAndSmoke': 31, 'fire': 109, 'neitherFireNorSmoke': 566, 'smoke': 79}
救回（火场近景）: 333，前缀 {'bothFireAndSmoke': 273, 'fire': 22, 'neitherFireNorSmoke': 38}
```

人工查看 `fasdd_check/tower_cam_samples.jpg`（应该都是瞭望塔视角）和 `rescued_samples.jpg`（应该都是夜间火场近景），确认后 `FIRE_FRAC_T = 0.02` 固定。

---

## 9. 构建干净数据集 FASDD_UAV_clean

处理流程：筛选 → 双阈值 pHash 聚类 → 以划分簇（阈值 8）为单位划分 train/val/test → 在子簇（阈值 4）内限量取样 → 负样本分为两档。

保存为 `/root/autodl-tmp/fasdd_scripts/build_clean_split.py`（最终版参数）：

```python
# build_clean_split.py：筛选 → 双阈值 pHash 聚类 → 按簇(THR)划分 → 按子簇(SUB_THR)取样 → 负样本分档
# DRY_RUN=True 时只打印统计；确认参数后改为 False 才会写文件（硬链接，不改原始数据）
import csv, json, random
from pathlib import Path
from collections import Counter, defaultdict
import numpy as np

ROOT  = Path("/root/autodl-tmp/FASDD_UAV_yolo")
CHECK = Path("/root/autodl-tmp/fasdd_check")
OUT   = Path("/root/autodl-tmp/FASDD_UAV_clean")
DRY_RUN = True

# ---- 参数：在看管线结果之前固定，写进论文 ----
THR = 8                              # 划分用：≤THR 同簇，跨划分距离必然 > THR（防泄漏）
SUB_THR = 4                          # 取样用：簇内再按 ≤SUB_THR 细分子簇（去冗余）
TEST_FRAC, VAL_FRAC = 0.30, 0.10     # 按簇分配（不是按图）
BIG_TO_TRAIN = 50                    # 超过该大小的簇只放 train
K_TRAIN_SUB = 2                      # train：每个子簇最多取几张正样本（负样本同理进候选池）
K_EVAL_SUB = 1                       # val/test：每个子簇最多取几张
EVAL_CAP_PER_CLUSTER = 5             # val/test：每个划分簇最多取几张，防止单个场景主导评估
MIN_SIDE_640 = 8                     # 含 640 尺度短边 < 8px 框的图整图删除
DROP_TOWER = True                    # 删除固定瞭望塔相机图（已人工核对）
NEG_RATIOS = (0.0, 0.10)             # train 负样本数 = 比例 × train 正样本数
SEED = 0
rng = random.Random(SEED)

PROPS_FILE = CHECK / "image_props_v2.csv"
MIN640_FILE = CHECK / "min_box_side640.csv"
PHASH_FILE = CHECK / "phash.npz"

z = np.load(PHASH_FILE, allow_pickle=True)
names, splits, H = z["names"], z["splits"], z["H"]
props = {(r["split"], r["image"]): r for r in csv.DictReader(open(PROPS_FILE))}
min640 = {(r["split"], r["stem"]): float(r["min_side_640"]) for r in csv.DictReader(open(MIN640_FILE))}

def is_pos(r): return int(r["n_fire"]) + int(r["n_smoke"]) > 0
def drop_reason(r):
    if DROP_TOWER and r["tower_cam"] == "1": return "tower_cam"
    if is_pos(r) and min640[(r["split"], Path(r["image"]).stem)] < MIN_SIDE_640: return "has_tiny_box"
    return None

# 1) 筛选
drop, drop_by_prefix, keep_idx = Counter(), defaultdict(Counter), []
for i, (s, n) in enumerate(zip(splits, names)):
    r = props[(s, n)]
    why = drop_reason(r)
    if why:
        drop[why] += 1
        drop_by_prefix[why][r["prefix"]] += 1
    else:
        keep_idx.append(i)
idx = np.array(keep_idx)
print(f"[筛选] 去掉 {dict(drop)}，保留 {len(idx)}/{len(names)}")
for why, c in drop_by_prefix.items():
    print(f"        {why} 按前缀: {dict(c)}")
def info(k): return splits[idx[k]], names[idx[k]]
def pos_k(k): return is_pos(props[info(k)])

# 2) 双阈值聚类（一次计算距离，同时更新两套并查集；SUB_THR 子簇必然嵌套在 THR 簇内）
POP = np.array([bin(i).count("1") for i in range(256)], dtype=np.uint8)
def ham(a, b):
    x = (a[:, None] ^ b[None, :]).view(np.uint8).reshape(len(a), len(b), 8)
    return POP[x].sum(-1, dtype=np.uint16)
Hs = H[idx]; n = len(Hs)
par_main, par_sub = np.arange(n), np.arange(n)
def find(p, i):
    while p[i] != i:
        p[i] = p[p[i]]; i = p[i]
    return i
for st in range(0, n, 500):
    b = np.arange(st, min(st + 500, n))
    D = ham(Hs[b], Hs)
    for thr, p in ((THR, par_main), (SUB_THR, par_sub)):
        ii, jj = np.nonzero(D <= thr)
        for a, c in zip(b[ii], jj):
            if a < c:
                ra, rc = find(p, a), find(p, c)
                if ra != rc: p[ra] = rc
root_main = np.array([find(par_main, k) for k in range(n)])
root_sub = np.array([find(par_sub, k) for k in range(n)])

groups = defaultdict(list)
for k in range(n): groups[root_main[k]].append(k)
cl = []
for gid, mem in groups.items():
    mem = sorted(mem, key=lambda k: info(k)[1])        # 按文件名排序，近似时间顺序
    pref = Counter(info(k)[1].split("_")[0] for k in mem).most_common(1)[0][0]
    cl.append(dict(id=int(gid), mem=mem, size=len(mem), prefix=pref))
sizes = np.array([c["size"] for c in cl])
print(f"[聚类] 划分簇(THR={THR}) {len(cl)} 个，最大 {sizes.max()}，>{BIG_TO_TRAIN} 的大簇 {(sizes > BIG_TO_TRAIN).sum()} 个；"
      f"子簇(SUB_THR={SUB_THR}) {len(set(root_sub.tolist()))} 个")

# 3) 按划分簇分配，按主前缀分层
assign, small = {}, defaultdict(list)
for c in cl:
    if c["size"] > BIG_TO_TRAIN: assign[c["id"]] = "train"
    else: small[c["prefix"]].append(c)
for p in sorted(small):
    lst = small[p]
    rng.shuffle(lst)
    nt, nv = round(len(lst) * TEST_FRAC), round(len(lst) * VAL_FRAC)
    for j, c in enumerate(lst):
        assign[c["id"]] = "test" if j < nt else ("val" if j < nt + nv else "train")

# 4) 按子簇取样
def pick(mem, k):
    if len(mem) <= k: return list(mem)
    step = len(mem) / k
    return [mem[int(i * step)] for i in range(k)]
def subgroups(mem):
    g = defaultdict(list)
    for k in mem: g[int(root_sub[k])].append(k)
    return [sorted(v, key=lambda k: info(k)[1]) for _, v in sorted(g.items())]

sel = defaultdict(list)
for c in sorted(cl, key=lambda c: c["id"]):
    a = assign[c["id"]]
    if a == "train":
        for sg in subgroups(c["mem"]):
            sel["train_pos"] += [(k, c["id"]) for k in pick([k for k in sg if pos_k(k)], K_TRAIN_SUB)]
            sel["train_neg_pool"] += [(k, c["id"]) for k in pick([k for k in sg if not pos_k(k)], K_TRAIN_SUB)]
    else:
        cand = [k for sg in subgroups(c["mem"]) for k in pick(sg, K_EVAL_SUB)]
        cand = sorted(cand, key=lambda k: info(k)[1])
        sel[a] += [(k, c["id"]) for k in pick(cand, EVAL_CAP_PER_CLUSTER)]
rng.shuffle(sel["train_neg_pool"])   # 固定顺序，neg00 ⊂ neg10，保证嵌套

# 5) 泄漏校验：val/test 到 train 全部成员的最小汉明距离（按构造应 > THR）
train_all = np.array([k for c in cl if assign[c["id"]] == "train" for k in c["mem"]])
leak = {}
for s in ("val", "test"):
    A = np.array([k for k, _ in sel[s]])
    if len(A) == 0:
        print(f"[泄漏校验] {s} 为空，跳过"); continue
    m = min(int(ham(Hs[A[i:i+500]], Hs[train_all]).min()) for i in range(0, len(A), 500))
    leak[s] = m
    print(f"[泄漏校验] {s} 到 train 的最小汉明距离 = {m}（应 > {THR}）{'  OK' if m > THR else '  !!! 异常'}")

# 6) 统计
def box_counts(items):
    f = s_ = p = 0
    for k, _ in items:
        r = props[info(k)]; f += int(r["n_fire"]); s_ += int(r["n_smoke"]); p += is_pos(r)
    return p, f, s_
def prefix_counts(items):
    return dict(Counter(info(k)[1].split("_")[0] for k, _ in items))

n_pos_train = len(sel["train_pos"])
for ratio in NEG_RATIOS:
    n_neg = min(round(ratio * n_pos_train), len(sel["train_neg_pool"]))
    p, f, s_ = box_counts(sel["train_pos"])
    print(f"[train neg{int(ratio*100):02d}] 正 {p} + 负 {n_neg} = {p + n_neg} 张；fire 框 {f}，smoke 框 {s_}")
print(f"        train 正样本前缀: {prefix_counts(sel['train_pos'])}；负样本池 {len(sel['train_neg_pool'])} 张")
for s in ("val", "test"):
    p, f, s_ = box_counts(sel[s])
    print(f"[{s}] {len(sel[s])} 张（正 {p}，负 {len(sel[s]) - p}）；fire 框 {f}，smoke 框 {s_}；"
          f"簇数 {len({g for _, g in sel[s]})}；前缀 {prefix_counts(sel[s])}")

# 7) 写文件
if not DRY_RUN:
    if OUT.exists(): raise SystemExit(f"{OUT} 已存在，确认后手动删除或换名")
    def link(items, img_dir, lbl_dir, manifest):
        img_dir.mkdir(parents=True); lbl_dir.mkdir(parents=True)
        with open(manifest, "w", newline="") as fp:
            w = csv.writer(fp); w.writerow(["image", "orig_split", "cluster", "subcluster", "is_positive"])
            for k, g in items:
                s, nm = info(k); stem = Path(nm).stem
                (img_dir / nm).hardlink_to(ROOT / "images" / s / nm)
                (lbl_dir / f"{stem}.txt").hardlink_to(ROOT / "labels" / s / f"{stem}.txt")
                w.writerow([nm, s, g, int(root_sub[k]), int(pos_k(k))])
    for s in ("val", "test"):
        link(sel[s], OUT / "eval" / "images" / s, OUT / "eval" / "labels" / s, OUT / f"manifest_{s}.csv")
    for ratio in NEG_RATIOS:
        tag = f"neg{int(ratio*100):02d}"
        n_neg = min(round(ratio * n_pos_train), len(sel["train_neg_pool"]))
        items = sel["train_pos"] + sel["train_neg_pool"][:n_neg]
        link(items, OUT / tag / "images" / "train", OUT / tag / "labels" / "train", OUT / f"manifest_{tag}.csv")
        (OUT / f"data_{tag}.yaml").write_text(
            f"path: {OUT}\ntrain: {tag}/images/train\nval: eval/images/val\ntest: eval/images/test\n"
            "names:\n  0: fire\n  1: smoke\n")
    (OUT / "params.json").write_text(json.dumps(dict(
        THR=THR, SUB_THR=SUB_THR, TEST_FRAC=TEST_FRAC, VAL_FRAC=VAL_FRAC, BIG_TO_TRAIN=BIG_TO_TRAIN,
        K_TRAIN_SUB=K_TRAIN_SUB, K_EVAL_SUB=K_EVAL_SUB, EVAL_CAP_PER_CLUSTER=EVAL_CAP_PER_CLUSTER,
        MIN_SIDE_640=MIN_SIDE_640, DROP_TOWER=DROP_TOWER, NEG_RATIOS=list(NEG_RATIOS), SEED=SEED,
        dropped=dict(drop), n_clusters=len(cl), n_subclusters=len(set(root_sub.tolist())),
        leak_min_hamming=leak, inputs=[str(PROPS_FILE), str(MIN640_FILE), str(PHASH_FILE)]),
        ensure_ascii=False, indent=2))
    print(f"已写出到 {OUT}")
```

### 9.1 先用 DRY_RUN 核对

```bash
python /root/autodl-tmp/fasdd_scripts/build_clean_split.py
```

预期输出：

```
[筛选] 去掉 {'has_tiny_box': 2155, 'tower_cam': 785}，保留 22157/25097
        has_tiny_box 按前缀: {'bothFireAndSmoke': 2112, 'fire': 36, 'smoke': 7}
        tower_cam 按前缀: {'bothFireAndSmoke': 31, 'fire': 109, 'neitherFireNorSmoke': 566, 'smoke': 79}
[聚类] 划分簇(THR=8) 1124 个，最大 5735，>50 的大簇 82 个；子簇(SUB_THR=4) 2219 个
[泄漏校验] val 到 train 的最小汉明距离 = 10（应 > 8）  OK
[泄漏校验] test 到 train 的最小汉明距离 = 10（应 > 8）  OK
[train neg00] 正 1423 + 负 0 = 1423 张；fire 框 1434，smoke 框 2375
[train neg10] 正 1423 + 负 142 = 1565 张；fire 框 1434，smoke 框 2375
        train 正样本前缀: {'bothFireAndSmoke': 588, 'smoke': 827, 'fire': 8}；负样本池 1046 张
[val] 141 张（正 101，负 40）；fire 框 97，smoke 框 217；簇数 104；前缀 {...}
[test] 429 张（正 317，负 112）；fire 框 380，smoke 框 <待补>；...
```

说明：
- `has_tiny_box` 是 2155 而不是第 7 步的 2211，因为先执行了瞭望塔规则，两条规则重叠的图只计入 `tower_cam`。
- test 的 smoke 框数在原始记录中被截断了。完成 9.2 后，请把实际数值补到本文档第 11 节。

### 9.2 正式写出

把脚本中的 `DRY_RUN = True` 改为 `DRY_RUN = False`，其他参数一律不改，然后运行：

```bash
python /root/autodl-tmp/fasdd_scripts/build_clean_split.py
```

---

## 10. 最终核对

```bash
cd /root/autodl-tmp/FASDD_UAV_clean
cat params.json
for d in neg00 neg10; do echo $d $(ls $d/images/train | wc -l) $(ls $d/labels/train | wc -l); done
for s in val test; do echo $s $(ls eval/images/$s | wc -l) $(ls eval/labels/$s | wc -l); done
```

预期输出：

```
neg00 1423 1423
neg10 1565 1565
val 141 141
test 429 429
```

`params.json` 中 `leak_min_hamming` 应为 `{"val": 10, "test": 10}`，`n_clusters` 为 1124，`n_subclusters` 为 2219。

---

## 11. 最终数据集说明

### 11.1 目录结构

```
FASDD_UAV_clean/
├── eval/
│   ├── images/val/     141 张（正 101，负 40）     用于挑选最佳权重
│   ├── images/test/    429 张（正 317，负 112）    用于报告最终结果
│   ├── labels/val/
│   └── labels/test/
├── neg00/images/train/ 1423 张（全部为正样本）
├── neg00/labels/train/
├── neg10/images/train/ 1565 张（1423 正 + 142 负）
├── neg10/labels/train/
├── data_neg00.yaml
├── data_neg10.yaml
├── manifest_{val,test,neg00,neg10}.csv
└── params.json
```

| 子集 | 图片 | 正样本 | 负样本 | fire 框 | smoke 框 |
|---|---|---|---|---|---|
| neg00 train | 1423 | 1423 | 0 | 1434 | 2375 |
| neg10 train | 1565 | 1423 | 142 | 1434 | 2375 |
| val | 141 | 101 | 40 | 97 | 217 |
| test | 429 | 317 | 112 | 380 | 待补 |

- neg00 ⊂ neg10：两档的正样本完全相同，唯一差别是 neg10 多了 142 张负样本。
- 所有实验共用同一个 eval（val/test）。
- 标签为 FASDD 原始人工标注，`0 = fire, 1 = smoke`，负样本对应空 txt。

### 11.2 yaml 示例（data_neg00.yaml）

```yaml
path: /root/autodl-tmp/FASDD_UAV_clean
train: neg00/images/train
val: eval/images/val
test: eval/images/test
names:
  0: fire
  1: smoke
```

### 11.3 manifest 字段

| 列 | 含义 |
|---|---|
| image | 文件名 |
| orig_split | 在 FASDD 官方划分中的归属 |
| cluster | 划分簇编号（pHash 阈值 8），同一簇必在同一划分 |
| subcluster | 子簇编号（pHash 阈值 4），即几乎一模一样的一组图 |
| is_positive | 人工标签是否非空 |

`is_positive` 来自人工标注，伪标签管线不能读取这一列。计算置信区间时，建议按 `cluster` 做 bootstrap。

---

## 12. 使用注意事项

1. **硬链接**：`FASDD_UAV_yolo` 和 `FASDD_UAV_clean` 中的文件，都是 `FASDD_UAV_raw` 原文件的硬链接。删除文件不影响原文件，但直接修改文件内容会同时改到原文件。不要在这些 `labels` 目录里写入或修改任何标签。
2. **伪标签**：伪标签要输出到单独的目录（例如 `/root/autodl-tmp/FASDD_UAV_pseudo/`），类别编号必须是 `0=fire, 1=smoke`。只需要在 `neg10/images/train` 上运行一次管线，neg00 所需的伪标签按文件名从中取出。
3. **参数冻结**：第 9 步的参数是在运行管线之前根据数据集统计确定的，之后不再修改。
4. **Ultralytics 缓存**：首次训练会在 labels 目录旁生成 `.cache` 文件，属于正常现象。
5. **人工基线训练示例**（超参数请换成实际使用的设置）：

```bash
cd /root/autodl-tmp/FASDD_UAV_clean
for tag in neg00 neg10; do
  for seed in 0 1 2; do
    yolo detect train data=data_${tag}.yaml model=yolov8s.pt imgsz=640 epochs=100 \
         seed=${seed} deterministic=True project=runs_fasdd name=human_${tag}_s${seed}
    yolo detect val data=data_${tag}.yaml split=test \
         model=runs_fasdd/human_${tag}_s${seed}/weights/best.pt name=test_human_${tag}_s${seed}
  done
done
```

---

## 附录 A：设计决策与理由

| 决策 | 理由 | 依据 |
|---|---|---|
| 不使用官方划分，重新划分 | 官方 val/test 中约 91% 的图与 train 近重复（pHash ≤ 4） | 第 5 步统计；与 Drones 2026 论文报告的 90.7% 一致 |
| 划分阈值 8，取样阈值 4 | 阈值 8 保证 val/test 到 train 的距离 > 8（实测为 10）；阈值 8 存在链式合并（最大簇 5735），所以在簇内按阈值 4 细分后再取样，保留多样性 | 第 5、9 步 |
| 大簇（> 50 张）全部放 train | 长视频序列不进入评估集，避免单一场景主导评估 | — |
| 删除 640 尺度下含短边 < 8 px 框的整张图 | 极小目标几乎全是火（fire 框中 12.8% 小于 8 px，smoke 框仅 0.1%）；整图删除而非只删框，避免模型检测到时被算作误检；阈值按"删除 ≤ 30% 则用 8 px"的预设规则确定 | 第 7 步 |
| 删除固定瞭望塔相机图 | 非无人机视角，目标又小又暗；用"四角暗 + 中心亮"初筛，再用火焰色占比 ≥ 0.02 救回夜间火场近景，经人工核对 | 第 6、8 步 |
| 保留大框（含整图框） | 大目标不影响判断，只是标注风格问题 | 人工决策 |
| 负样本分 neg00 和 neg10 两档 | 原始数据中负样本约占 48%；两档对比可以量化负样本对伪标签管线的影响 | 第 3 步 |
| TEST_FRAC = 0.30 | 0.20 时 test 正样本只有 213 张，低于预设的 250 张；调到 0.30 后为 317 张 | 第 9 步两次 DRY_RUN |

## 附录 B：已知局限

- pHash 只能识别像素层面的近重复。同一场火从不同角度拍摄的画面，仍可能分别出现在 train 和 test 中。
- 标注风格不统一（火框粒度、整图烟框）的问题，重新划分无法解决，需要结合风格容忍指标或重新标注 test 来处理。
- 纯火场景极少（train 8 张，test 1 张），FASDD 上的 fire 类结果主要反映火烟共存的场景。
- 去重后有效规模约为 2,200 个独立画面，test 正样本 317 张，建议跑多个随机种子，并按簇做 bootstrap 给出置信区间。
