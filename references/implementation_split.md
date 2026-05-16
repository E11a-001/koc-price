# KOC Pricing Skill: 代码化与 AI 判断拆分

本文件用于把 `koc-pricing` skill 产品化、脚本化或接入自动化工作流时的实现边界定义清楚。

核心原则：

**有公式、有阈值、有映射表的部分，优先代码化。**

**需要语义理解、内容判断、产品匹配解释的部分，交给 AI。**

**先分清报价口径，再计算价格。**

---

## 0. 先判断报价口径

实现时，先确定当前任务属于哪一种价格口径：

- `strict_cpm_cap`
- `fair_market_range`
- `creator_ask_floor_range`

推荐判定规则：

- 用户问“最多多少钱”“值不值”“上限多少”“按 CPM 算能给多少” -> `strict_cpm_cap`
- 用户问“市场上一般多少钱能成交” -> `fair_market_range`
- 用户问“博主会报多少”“为什么对方报这么贵” -> `creator_ask_floor_range`

YouTube 默认 strict cap 建议：

- 若用户未单独指定，且内容为设计 / AI / 工具 / 创意教育等高意图垂类，可先用 **CPM 60–80**
- `strict_cpm_cap` 不得因为 floor price 自动上调

---

## 1. 必须代码化的部分

以下内容不得交给 AI 自由估算，必须使用规则或代码实现：

- 平台识别：YouTube / Instagram / X
- URL 标准化与 handle 提取
- 原始数字清洗：`1.1k -> 1100`
- 时间窗口过滤：如近 30 天、近 90 天
- 播放量中位数、平均值、分位数
- 异常值检测与排除规则
- 互动率计算
- 更新频率判断
- 粉丝量级分层：Nano / Micro / Mid / Macro / Mega
- 理论报价公式
- 报价口径分流
- Floor Price 兜底逻辑
- Bundle 折扣逻辑
- CAC / ROI 计算
- 规则明确的评分映射

建议写成独立函数，例如：

- `parse_count(text)`
- `filter_posts_by_days(posts, days)`
- `calc_median_views(posts)`
- `calc_avg_views(posts)`
- `calc_engagement_rate(followers, avg_likes, avg_comments)`
- `classify_follower_tier(n)`
- `classify_activity(posts)`
- `apply_floor_price(platform, price, tier)`
- `select_quote_mode(user_intent)`
- `calc_youtube_strict_cap(views, cpm_low, cpm_high, format_coeff, geo_coeff)`
- `bundle_price(main_price, second_price, third_price)`
- `calc_cac(cost, impressions, ctr, cvr)`
- `calc_roi(cost, conversions, ltv)`

---

## 2. 适合 AI 判断的部分

以下内容更适合由 AI 基于上下文做判断，并输出简短理由：

- Niche Match：创作者受众与产品是否高度匹配
- Content Quality：内容是否清晰、专业、适合承接广告
- 主力平台推荐：为什么 YouTube / Instagram / X 更适合
- 风险判断：受众偏离、流量衰退、平台不适配
- 谈判建议与压价策略
- 数据不足时的保守解释

建议 AI 输出结构化结果：

```json
{
  "score": 25,
  "label": "high_match",
  "reason": "受众为设计师和创意工作者，与 AI 设计工具高度匹配"
}
```

---

## 3. 半代码半 AI 的部分

以下内容建议采用“代码先算，AI 再解释”的模式：

### 账号质量总分

- `互动率`：代码打分
- `更新频率`：代码打分
- `流量稳定性`：代码计算波动，AI 解释
- `受众匹配度`：AI 打分
- `内容质量`：AI 打分
- `总分`：代码汇总

### 主力平台判断

- 平台阈值和候选条件，可由规则先筛
- 最终推荐理由，由 AI 结合产品形态解释

---

## 4. 数据缺失时的降级策略

只要缺少关键指标，必须明确标记置信度，不得把估算当成精确值。

### YouTube

- 如果能拿到近 90 天视频列表和播放量：`high`
- 如果只能拿到部分近期视频：`medium`
- 如果只有订阅量，没有近期播放：`low`

### Instagram

- 如果拿到近期 10 条内容的点赞/评论均值：`high`
- 如果只有粉丝量，没有互动数据：`low`
- 如果第三方平台给出冲突的 ER：标记 `needs verification`

### X

- 如果能拿到近期 Thread/Post 阅读量与互动量：`high`
- 如果只有粉丝量和 bio：`low`

最终报告中建议增加：

- `Data Confidence`
- `Exact Metrics`
- `Estimated Metrics`
- `Needs Verification`

---

## 5. 异常值与稳定性建议

### YouTube

- 若某条视频播放量 `> 其他近期视频平均值 5 倍`，视为潜在爆款
- 如存在明显爆款，优先使用**中位数**而非平均数

### Instagram / X

- 若近 10 条内容互动差异极大，稳定性下调
- 若只有少数高互动内容、其余表现弱，避免直接用 Top Post 推定整体报价

---

## 6. 推荐的数据流

```text
links
 -> platform detection
 -> raw fetch
 -> normalization
 -> deterministic metrics
 -> rule-based scoring
 -> AI judgement
 -> pricing
 -> bundle pricing
 -> report assembly
```

推荐拆为以下模块：

- `fetchers/`
- `normalizers/`
- `metrics/`
- `pricing/`
- `scoring/`
- `llm/`
- `reports/`

建议 `reports/` 默认输出结构化对象，再由渲染层决定是否转成 YAML / JSON / Markdown。

推荐最小输出对象：

- `creator`
- `analysis_context`
- `quality_score`
- `performance_metrics`
- `pricing`
- `bundle_recommendation`
- `confidence`
- `recommendation`

---

## 7. 最重要的实现约束

- 不要让 AI 直接计算中位数、ER、CAC、ROI
- 不要让 AI 自由决定 floor price
- 不要把 `strict_cpm_cap` 和 `fair_market_range` 混成一个数字
- 不要只看粉丝量报价
- 不要在关键指标缺失时给出高置信度报价
- 不要让非原生平台（如 TikTok）替代本 skill 的主定价逻辑，除非新增对应 reference 和公式
