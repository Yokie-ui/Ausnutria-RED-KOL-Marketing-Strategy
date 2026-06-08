# DuckDB 数据字典：post_gpt_tags_pgy_price

## 基本信息

| 项目 | 说明 |
|------|------|
| 库文件 | `output/joined/post_gpt_tags_pgy_price.duckdb` |
| 表名 | `post_gpt_tags_pgy_price` |
| 行数 | 271,452（以实际查询为准） |
| 列数 | 30 |
| 构建脚本 | `scripts/join_gpt_tags_pgy_price.py` |
| 分类 SQL | `scripts/sql/temp_20260602_gpt_rule_content_brand_tag.sql` |
| 品牌归一 | `scripts/brand_norm.py` |
| 蒲公英数据 | `output/pgy_kol_price/part0001～0007`（`fetch_status=ok`） |

## 数据链路

```
Iceberg 主贴表 (dwd_brand_sentiment_ice_red_book_post_data)
  → GPT 规则：内容形态 + 品牌打标 (Hive/Kyuubi)
  → 左关联 uid
蒲公英报价 (pgy_kol_price, fetch_status=ok)
  → DuckDB 本地表
```

**Hive 拉数筛选条件**

- `partition_industry = 'temp_20260602'`
- `fans_cnt > 200`

**关联方式**

- 关联键：主贴 `uid` = 蒲公英 `query_uid`
- 关联类型：左关联（无蒲公英数据时 `pgy_*` 为 NULL）

**说明**

- 导出 DuckDB 时**未包含** `text_all` 列（标题 + 正文 + ASR 拼接全文），分类结果已写入 `content_category` / `brand_*` 等字段。
- 分类所用文本域与 SQL 一致：`title ### content ### audio_asr_content`。

---

## 字段说明（全表）

### 1. 主贴标识与作者

| 字段 | 类型 | 说明 | 示例/备注 |
|------|------|------|-----------|
| `id` | VARCHAR | 主贴在数仓中的内部主键 ID | — |
| `item_id` | VARCHAR | 小红书笔记 ID，业务上唯一标识一篇帖子 | — |
| `uid` | VARCHAR | 发帖博主用户 ID；与蒲公英查询用的 `query_uid` 一致 | 用于关联报价 |
| `author` | VARCHAR | 博主昵称（展示名） | 来自主贴表 |
| `fans_cnt` | BIGINT | 主贴侧粉丝数（数仓快照） | 本批筛选 `> 200` |
| `interaction_cnt` | BIGINT | 互动量汇总（点赞、收藏、评论等，口径以源表为准） | — |
| `publish_time` | BIGINT | 发布时间，**毫秒级 Unix 时间戳** | 可用 `epoch_ms(publish_time)` 转日期 |

### 2. 分区字段

| 字段 | 类型 | 说明 | 示例/备注 |
|------|------|------|-----------|
| `partition_industry` | VARCHAR | 行业分区 | 本批为 `temp_20260602` |
| `partition_task` | VARCHAR | 任务/批次分区 | 标识采集或任务维度 |
| `partition_day` | BIGINT | 按日分区键 | 通常为 `yyyyMMdd` 整数 |

### 3. 文本内容

| 字段 | 类型 | 说明 | 示例/备注 |
|------|------|------|-----------|
| `title` | VARCHAR | 笔记标题 | 已去除换行符 |
| `content` | VARCHAR | 笔记正文 | 可能含话题、emoji、长段落 |
| `audio_asr_content` | VARCHAR | 视频笔记语音转写（ASR） | 无视频或无声时可能为空 |

### 4. 内容形态分类（GPT 规则）

基于关键词 `RLIKE` 匹配，**先命中先归类、单帖单类**。规则见 `scripts/sql/temp_20260602_gpt_rule_content_brand_tag.sql` 与 `output/gpt.txt`。

| 字段 | 类型 | 说明 | 示例/备注 |
|------|------|------|-----------|
| `content_category` | VARCHAR | **细分类**（约 16+ 类） | 见下方「content_category 取值」 |
| `content_category_l1` | VARCHAR | **一级汇总类**（约 10 类） | 与 `gpt.txt` 报告口径对齐，规则略粗 |
| `has_content_tag` | BIGINT | 是否命中非「其他」内容类 | `content_category <> '其他'` 时为 `1`，否则 `0` |

#### content_category 主要取值

| 取值 | 含义概要 |
|------|----------|
| 待产包/孕期 | 待产包、孕晚期、月子中心、第一口奶等 |
| 品牌活动/线下活动 | 妈妈班、星妈会、研学、沙龙、线下报名等 |
| 补贴促销/门店导购 | 育儿补贴、领奶粉、新客、顾问、优惠券等 |
| 购物晒单/囤货 | 囤货、晒单、香港购物、开箱等 |
| 敏感肠胃/特殊需求 | 敏宝、水解、过敏、玻璃胃等 |
| 羊奶粉 | 羊乳、羊奶粉品类及相关品牌词 |
| 有机奶粉 | 有机认证、有机牧场等 |
| HMO奶粉 | HMO、母乳低聚糖相关 |
| A2奶粉 | A2 蛋白赛道（与品牌 a2 可并存） |
| 长肉追高 | 追肉、追高、儿保、体重等 |
| 自护力/免疫力 | 自护力、乳铁蛋白（非测评语境）等 |
| 奶粉测评/对比 | 测评、横评、PK、选奶攻略等 |
| 选奶求助/推荐 | 求推荐、好纠结、选哪个等 |
| 单品种草/使用反馈 | 种草、空瓶、回购、亲测等 |
| 育儿生活场景 | 幼儿园、露营、遛娃、成长记录等 |
| 喂养知识/科普 | 转奶、冲泡、水奶、喂奶科普等 |
| 其他 | 未命中以上规则 |

#### content_category_l1 主要取值

待产包/孕期、品牌活动/线下活动、补贴促销/门店导购、购物晒单/囤货、敏感肠胃/特殊需求、奶粉测评/对比、选奶求助/推荐、单品种草/使用反馈、育儿生活场景、喂养知识/科普、其他。

> 细分类中的羊奶粉、HMO、A2 等品类在 L1 中可能归入「其他」。

### 5. 品牌打标

| 字段 | 类型 | 说明 | 示例/备注 |
|------|------|------|-----------|
| `brand_primary` | VARCHAR | **主品牌/产品线**（单值） | 约 39 个取值；CASE 优先级取首个命中 |
| `brand_tags` | VARCHAR | **多品牌命中**（逗号拼接） | 用于共现分析；可与 `brand_primary` 不同 |
| `has_brand_tag` | BIGINT | 是否有主品牌 | `brand_primary IS NOT NULL` 时为 `1` |
| `brand_primary_norm` | VARCHAR | **母品牌归一** | 见下方「brand_primary_norm 映射」 |

#### brand_primary 示例（子品牌/产品线，节选）

皇家美素佳儿、美素佳儿源悦、飞鹤星飞帆/卓睿、爱他美卓傲、爱他美海外版、a2至初、a2紫白金、惠氏启赋、合生元派星、金领冠 等。

#### brand_primary_norm 映射（母品牌）

| brand_primary_norm | 归并自 brand_primary（节选） |
|--------------------|------------------------------|
| 美素佳儿 | 皇家美素佳儿、皇家美素佳儿旺玥、美素佳儿、美素佳儿源悦 |
| 飞鹤 | 飞鹤、飞鹤星飞帆/卓睿、飞鹤臻稚卓蓓 |
| a2 | a2、a2至初、a2紫曜、a2紫白金、a2臻智护 |
| 惠氏 | 惠氏启赋、惠氏S-26 |
| 爱他美 | 爱他美、领熠、卓傲、亲熠、羊奶粉、海外版 |
| 合生元 | 合生元、合生元派星 |
| 伊利 | 金领冠、伊利QQ星 |
| 雀巢 | 雀巢/beba |
| 君乐宝、美赞臣、贝拉米、海普诺凯、佳贝艾特 等 | 与原名一致或见 `scripts/brand_norm.py` |

### 6. 蒲公英报价（左关联，`pgy_` 前缀）

数据来源：蒲公英开放平台 `POST /api/open/pgy/kol/data/detail`，脚本 `scripts/batch_pgy_kol_by_uid.py`。

| 字段 | 类型 | 说明 | 示例/备注 |
|------|------|------|-----------|
| `pgy_kol_nick_name` | VARCHAR | 蒲公英侧博主昵称 | — |
| `pgy_red_id` | VARCHAR | 小红书号（red_id） | — |
| `pgy_kol_fan_num` | DOUBLE | **蒲公英接口粉丝数** | 粉丝分层统计用；与 `fans_cnt` 口径/时点可能不同 |
| `pgy_kol_id` | VARCHAR | 蒲公英达人 ID | 非空表示接口命中达人 |
| `pgy_kol_operate_state` | DOUBLE | 达人运营状态（接口数值编码） | 以蒲公英文档为准 |
| `pgy_kol_credit_level` | DOUBLE | 达人信用等级（接口数值） | — |
| `pgy_mcn_name` | VARCHAR | 所属 MCN 名称 | 无则 NULL |
| `pgy_video_price` | DOUBLE | 视频笔记报价（元） | 无报价为 NULL |
| `pgy_price` | DOUBLE | 图文/默认笔记报价（元） | — |
| `has_pgy_price` | BIGINT | 是否关联到有效蒲公英数据 | `pgy_kol_id` 非空为 `1` |

#### 粉丝分层参考（基于 `pgy_kol_fan_num`）

| 分层标签 | 条件 |
|----------|------|
| `>500001` | `> 500001` |
| `300001-500000` | `300001 ~ 500000` |
| `200001-300000` | `200001 ~ 300000` |
| `100001-200000` | `100001 ~ 200000` |
| `50001-100000` | `50001 ~ 100000` |
| `10001-50000` | `10001 ~ 50000` |
| `5001-10000` | `5001 ~ 10000` |
| `1000-5000` | `1000 ~ 5000` |

---

## 字段分组速查

| 分组 | 字段列表 |
|------|----------|
| 帖子/作者 | `id`, `item_id`, `uid`, `author`, `fans_cnt`, `interaction_cnt`, `publish_time` |
| 分区 | `partition_industry`, `partition_task`, `partition_day` |
| 文本 | `title`, `content`, `audio_asr_content` |
| 内容分类 | `content_category`, `content_category_l1`, `has_content_tag` |
| 品牌 | `brand_primary`, `brand_primary_norm`, `brand_tags`, `has_brand_tag` |
| 蒲公英 | `pgy_kol_nick_name`, `pgy_red_id`, `pgy_kol_fan_num`, `pgy_kol_id`, `pgy_kol_operate_state`, `pgy_kol_credit_level`, `pgy_mcn_name`, `pgy_video_price`, `pgy_price`, `has_pgy_price` |

---

## 常用 SQL 示例

```sql
-- 查看表结构
DESCRIBE post_gpt_tags_pgy_price;

-- 有品牌 + 有蒲公英报价
SELECT COUNT(*) FROM post_gpt_tags_pgy_price
WHERE has_brand_tag = 1 AND has_pgy_price = 1;

-- 品牌 × 内容形态分布
SELECT brand_primary_norm, content_category_l1, COUNT(*) AS cnt
FROM post_gpt_tags_pgy_price
WHERE brand_primary_norm IS NOT NULL
GROUP BY 1, 2
ORDER BY cnt DESC;

-- 发布时间转日期
SELECT DATE_TRUNC('day', epoch_ms(publish_time)) AS pub_date, COUNT(*) AS cnt
FROM post_gpt_tags_pgy_price
GROUP BY 1
ORDER BY 1;

-- 粉丝分层（蒲公英粉丝）
SELECT
    brand_primary_norm,
    CASE
        WHEN pgy_kol_fan_num > 500001 THEN '>500001'
        WHEN pgy_kol_fan_num BETWEEN 300001 AND 500000 THEN '300001-500000'
        WHEN pgy_kol_fan_num BETWEEN 10001 AND 50000 THEN '10001-50000'
        WHEN pgy_kol_fan_num BETWEEN 1000 AND 5000 THEN '1000-5000'
    END AS fans_bucket,
    COUNT(*) AS cnt
FROM post_gpt_tags_pgy_price
WHERE has_pgy_price = 1 AND brand_primary_norm IS NOT NULL
GROUP BY 1, 2;
```

---

## 关联文件

| 文件 | 用途 |
|------|------|
| `output/joined/brand_fans_content_l1_stats.csv` | 品牌 × 粉丝层 × content_category_l1 统计明细 |
| `output/joined/brand_fans_bucket_overview.csv` | 各品牌各粉丝层样本量 |
| `output/joined/brand_fans_content_l1_report.docx` | 分析结论文档 |
| `scripts/stats_brand_fans_content_l1.py` | 重跑粉丝分层统计 |
| `scripts/normalize_brand_primary_duckdb.py` | 重算 `brand_primary_norm` |

---

*文档生成说明：字段定义与 `join_gpt_tags_pgy_price.py`、`temp_20260602_gpt_rule_content_brand_tag.sql`、`brand_norm.py` 保持一致。*
