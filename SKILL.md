---
name: koc-pricing
description: KOC/KOL 赞助定价评估与 ROI 计算。当用户提供 YouTube、Instagram 或 X (Twitter) 的博主链接，需要评估合理合作报价区间、计算预估获客成本（CAC）或 ROI 时使用。支持多平台综合评估，核心功能是：输入博主链接 → 评估账号质量 → 输出主力平台推荐与打包报价区间。
---

# KOC/KOL 赞助定价 Skill

## 核心原则

**订阅量/粉丝量不是定价依据，真实曝光量和互动率才是。**

三个平台的定价逻辑各不相同：
- **YouTube**：按播放量中位数定价（详见 `references/youtube_pricing.md`）
- **Instagram**：按粉丝量级 + 互动率定价（详见 `references/instagram_pricing.md`）
- **X (Twitter)**：按受众专业属性 + 内容格式定价（详见 `references/x_pricing.md`）

在开始分析前，先读取对应平台的 reference 文件。

## 报价口径选择（必须）

在给出任何报价前，必须先判断用户要的是哪一种价格口径，尤其是 **YouTube**：

- **Strict CPM Cap**：品牌方从 ROI / 采购上限视角出价，只按真实曝光和格式系数计算，**不得**自动抬到创作者底价
- **Fair Market Range**：品牌方合理支付区间，可以参考市场成交带和 floor price
- **Creator Ask / Floor Range**：创作者可能会报或能接受的价格带，主要反映制作成本和保留价

**默认规则：**
- 如果用户问的是“最多多少钱”“上限多少”“按 CPM 算值多少”“ROI 值不值”，优先输出 **Strict CPM Cap**
- 如果用户问的是“市场上大概多少钱能成交”，输出 **Fair Market Range**
- 如果用户问的是“博主可能会开什么价”或“为什么对方报这么高”，输出 **Creator Ask / Floor Range**

**YouTube 默认 strict cap 参数：**
- 如果用户没有单独指定 CPM，且频道属于高意图工具/设计/AI/创意教育类，可先用 **CPM 60–80**
- `Dedicated Video` 仍按格式系数 `1.3x – 1.5x` 计算
- `Strict CPM Cap` 是品牌侧 ceiling，不得因为 floor table 自动上调

如果任务不是“手动评估报价”，而是要**把本 skill 程序化、做成工具或接入工作流**，先额外读取 `references/implementation_split.md`。该文件定义了：
- 哪些指标必须代码化，不能交给 AI 自由发挥
- 哪些部分可以交给 AI 做语义判断和解释
- 数据缺失时的降级策略
- 推荐的数据流和函数边界

**基础范围说明：**
- 本 skill 的原生定价范围仅覆盖 **YouTube / Instagram / X**
- 如果用户额外提供 TikTok 或其他平台链接，可作为辅助判断创作者体量和分发能力的信号
- 除非单独实现该平台的 reference 与公式，否则不得让非原生平台替代本 skill 的主报价逻辑

---

## 账号质量评分标准（满分 100 分）

在给出报价前，必须先对账号质量进行评分，判断是否值得合作。

| 评估维度 | 评分标准 | 权重 |
|---------|---------|------|
| **互动率 (Engagement Rate)** | 极高(>6%): 30分 / 优质(3-6%): 25分 / 正常(1.5-3%): 15分 / 偏低(0.5-1.5%): 5分 / 危险(<0.5%): 0分 (僵尸粉警告) | 30% |
| **更新频率 (Activity)** | 活跃(每周更新): 20分 / 正常(每月更新): 15分 / 停更(>2个月未更新): 0分 | 20% |
| **受众匹配度 (Niche Match)** | 高度匹配(如AI推AI): 25分 / 中等匹配(如科技推AI): 15分 / 低匹配(如旅行推AI): 5分 | 25% |
| **内容质量 (Content Quality)** | 制作精良/逻辑清晰: 15分 / 普通: 10分 / 粗糙: 5分 | 15% |
| **真实流量稳定性** | 播放量/点赞量稳定: 10分 / 极度依赖偶发爆款: 5分 | 10% |

**合作建议阈值：**
- **> 80分**：强烈推荐合作，可接受溢价
- **60 - 80分**：值得合作，按标准区间压价
- **< 60分**：不建议合作，除非纯 CPA 模式或极低白菜价

---

## 多平台综合评估与报价流程（Step-by-Step）

当用户提供一个或多个平台的链接时，按以下步骤操作：

### Step 0：先确定报价口径

在正式计算前，先确定本次回答要优先输出哪种价格：

- `strict_cpm_cap`
- `fair_market_range`
- `creator_ask_floor_range`

如果用户没有明确说明，且问题带有明显采购/压价/ROI 视角，默认主输出 `strict_cpm_cap`，其他两档可作为补充参考。

### Step 1：分别采集各平台数据

**YouTube：**
- 订阅量
- 近 90 天播放量中位数（排除异常爆款）
- 频道 Niche 和受众地区

**Instagram：**
- 粉丝量
- 近期 Reels/Posts 的平均点赞数和评论数
- 计算互动率 = (平均点赞 + 平均评论) ÷ 粉丝数

**X (Twitter)：**
- 粉丝量
- 近期 Thread/Post 的平均阅读量和互动量
- 判断受众专业属性（Tech/AI/B2B 等）

**如果是在程序中实现本步骤：**
- 时间窗口筛选、异常值排除、平均值/中位数、互动率、更新频率，必须由代码完成
- Niche 识别、受众匹配度、内容质量、主力平台解释，可由 AI 参与判断
- 只要某个关键指标无法稳定抓取，必须在最终报告中标记为 `estimated` 或 `needs verification`

### Step 2：账号质量评分与主力平台判断

1. 根据上述评分标准，为每个平台打分。
2. **判断主力平台**：
   - **YouTube 为主力**：如果 YouTube 播放量中位数 > 1,000，且产品需要深度演示（如复杂 AI 工具）。
   - **Instagram 为主力**：如果 Instagram 互动率 > 3%，且产品适合视觉种草（如 C 端 App）。
   - **X 为主力**：如果 X 账号属于 Tech/AI/B2B 垂类，且产品面向专业人群（如开发者工具）。

**实现约束：**
- `互动率`、`更新频率`、`流量稳定性` 的分值应优先由规则/代码直接映射
- `受众匹配度`、`内容质量` 可以由 AI 打分，但必须附简短理由
- `总分` 可由代码汇总，不能让 AI 自由心算

### Step 3：分别计算单平台理论报价

参考 `references/` 目录下的具体定价逻辑：
- **YouTube**：(播放量中位数 ÷ 1000) × Niche CPM × 格式系数 × 地区系数
- **Instagram**：基础格式费率 × 互动率系数 × Niche 溢价系数
- **X**：基础格式费率 × Niche 溢价系数

*注意：必须对照各平台的心理底价（Floor Price），理论值低于底价时，以底价为准。*

**YouTube 特别说明：**
- 上面这句 `以底价为准` 只适用于 `fair_market_range` 或 `creator_ask_floor_range`
- 如果当前口径是 `strict_cpm_cap`，**不得**因为 floor price 自动上调报价
- 因此 YouTube 至少应区分：
  - `Strict CPM Cap`
  - `Fair Market Range`
  - `Creator Ask / Floor Range`

### Step 4：多平台打包定价逻辑

如果用户希望多平台打包合作，适用以下折扣规则：

1. **确定核心 Deliverable**：以主力平台的内容为核心（如 YouTube 专属视频）。
2. **附加平台作为分发渠道**：其他平台仅作为内容分发（如 IG Reel 剪辑版、X Thread 转发）。
3. **打包折扣计算**：
   - 主力平台：100% 报价
   - 第二平台：单平台报价的 50% - 70%（因为内容复用，创作者生产成本降低）
   - 第三平台：单平台报价的 30% - 50%
   - **总打包价 = 主力平台价 + 第二平台折后价 + 第三平台折后价**

### Step 5：输出结构化结果（默认）

默认输出 **结构化 YAML**，必要时再在 YAML 后追加一小段 `summary` 解释。不要只输出自由 prose。

#### 输出规则

- 默认格式：`yaml`
- 所有金额字段统一用 `usd`
- 能精确计算的数值用数字，不要写模糊文本
- 推断值允许写在 `estimated_metrics`、`assumptions` 或 `notes`
- 如果是 **YouTube**，必须拆出：
  - `strict_cpm_cap`
  - `fair_market_range`
  - `creator_ask_floor_range`

#### 推荐 YAML 模板

```yaml
creator:
  name: "[creator name]"
  platforms:
    - "[youtube|instagram|x]"
  urls:
    - "[profile url]"
  primary_platform: "[youtube|instagram|x]"

analysis_context:
  quote_mode_primary: "[strict_cpm_cap|fair_market_range|creator_ask_floor_range]"
  product_niche: "[product niche or unknown]"
  assumptions:
    strict_cpm_range: "[60-80 or custom]"
    geo_multiplier: "[1.0 or estimated value]"
    niche_cpm_basis: "[which niche table was used]"

quality_score:
  total: 0
  verdict: "[strong_recommend|worth_testing|not_recommended]"
  breakdown:
    engagement: 0
    activity: 0
    niche_match: 0
    content_quality: 0
    stability: 0

performance_metrics:
  sample_window_days: 90
  sampled_posts: 0
  median_views: null
  average_views: null
  engagement_rate: null
  outlier_signal:
    has_outlier: false
    reason: "[short reason]"

pricing:
  youtube:
    strict_cpm_cap:
      integration_60_90s:
        min_usd: 0
        max_usd: 0
        formula: "[formula]"
      dedicated_video:
        min_usd: 0
        max_usd: 0
        formula: "[formula]"
        recommended_ceiling_usd: 0
    fair_market_range:
      integration_60_90s:
        min_usd: 0
        max_usd: 0
      dedicated_video:
        min_usd: 0
        max_usd: 0
    creator_ask_floor_range:
      integration_60_90s:
        min_usd: 0
        max_usd: 0
      dedicated_video:
        min_usd: 0
        max_usd: 0
  instagram:
    reel:
      min_usd: 0
      max_usd: 0
    post:
      min_usd: 0
      max_usd: 0
  x:
    thread:
      min_usd: 0
      max_usd: 0
    single_post:
      min_usd: 0
      max_usd: 0

bundle_recommendation:
  deliverables:
    - "[main deliverable]"
  total_min_usd: 0
  total_max_usd: 0
  pricing_logic: "[short explanation]"

confidence:
  data_confidence: "[high|medium|low]"
  exact_metrics:
    - "[metric]"
  estimated_metrics:
    - "[metric]"
  needs_verification:
    - "[metric]"

recommendation:
  rationale: "[short reason]"
  negotiation_advice:
    - "[advice]"
  risks:
    - "[risk]"
```

#### 可选附加总结

如果用户是人类阅读场景，可以在 YAML 后补一段简短总结：

```markdown
summary: |
  [2-4 句高层结论，说明这个号值不值得合作、严格上限在哪、市场成交价大概在哪。]
```
