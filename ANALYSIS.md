# Solana 链上套利 Bot 合约 - 深度代码分析

## 一、项目概览

| 项目 | 详情 |
|------|------|
| **合约名称** | `arb_touyi` |
| **Program ID** | `DxeQQ7PQ94j26ism5ivTqNHAkteFNmgRpqYx7XQFqs9Z` |
| **框架** | Anchor 0.30.1 |
| **语言** | Rust (on-chain) |
| **目标网络** | Solana Mainnet |
| **核心功能** | 多DEX跨市场自动套利 |
| **收费地址** | `B2kcKQCZUWvK59w9V9n7oDiFwqrh5FowymgpsKZV5NHu` (10%利润抽成) |

---

## 二、架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                    lib.rs (入口层)                           │
│  arb_process_64_account / arb_process_32_account            │
│  test_raydium_clmm                                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│                processor.rs (核心引擎)                       │
│  main_process → create_market_list → process_arb → try_arb  │
│                                       ↓                     │
│                              market_arb_calc (数学计算)      │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│              base_market.rs (Market抽象层)                    │
│  trait BaseMarketPool / trait CreateMarket                    │
│  enum MarketType: NormalAMM / NormalCMM / NormalCLMM         │
└──────────────────────┬──────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────┐
│              markets/ (6种DEX具体实现)                        │
│  ┌─────────────────┐ ┌────────────────┐ ┌────────────────┐  │
│  │ Meteora DLMM    │ │ Raydium AMM    │ │ PumpSwap       │  │
│  │ (type=0, CMM)   │ │ (type=1, AMM)  │ │ (type=2, AMM)  │  │
│  └─────────────────┘ └────────────────┘ └────────────────┘  │
│  ┌─────────────────┐ ┌────────────────┐ ┌────────────────┐  │
│  │ Raydium CLMM    │ │ Raydium CPMM   │ │ Meteora AMM    │  │
│  │ (type=3, CLMM)  │ │ (type=4, AMM)  │ │ (type=5, AMM)  │  │
│  └─────────────────┘ └────────────────┘ └────────────────┘  │
│  ┌─────────────────┐                                         │
│  │ MockReverse     │ (方向翻转代理)                           │
│  └─────────────────┘                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、支持的 DEX 及市场类型分类

### 3.1 市场类型枚举

| MarketType | 含义 | 适用DEX |
|-----------|------|---------|
| `NormalAMM` | 恒定乘积做市商 (x*y=k) | Raydium AMM, Raydium CPMM, PumpSwap, Meteora AMM |
| `NormalCMM` | 集中流动性做市商 (Bin-based) | Meteora DLMM |
| `NormalCLMM` | 集中流动性做市商 (Tick-based) | Raydium CLMM |

### 3.2 各 DEX 详细分析

#### (1) Meteora DLMM (type=0) — `meteora_market.rs`
- **Market类型**: `NormalCMM`
- **消耗账户数**: 10个
- **核心特点**:
  - 基于 Bin 的集中流动性模型
  - 价格公式: `price = (10000 / (10000 + bin_step))^active_id`
  - 费率 = base_fee + variable_fee (含波动率累加器)
  - 支持最多3个 bin_array
  - 通过原始字节偏移解析链上数据 (offset: 8+32+32+1+2+1 等)
  - swap 使用 CPI 调用，指令discriminator: `f8c69e91e17587c8`

#### (2) Raydium AMM (type=1) — `raydium_market.rs`
- **Market类型**: `NormalAMM`
- **消耗账户数**: 5个
- **核心特点**:
  - 经典 x*y=k 恒定乘积模型
  - 从 amm_info 解析 fee_numerator/fee_denominator (offset 128+48)
  - 池子余额 = token_account余额 - need_take_pnl (offset 192)
  - swap 指令: opcode `9` + amount_in + min_out
  - account_meta 中多处填充 amm_info 占位 (为兼容 Serum/OpenBook market)

#### (3) PumpSwap (type=2) — `pumpswap_market.rs`
- **Market类型**: `NormalAMM`
- **消耗账户数**: 11个
- **核心特点**:
  - fee_denominator 固定为 10000
  - 费率 = base_fee + protocol_fee + creator_fee (三层费率)
  - 买入(y→x)使用 discriminator `66063d1201daebea`，需预计算 amount_out
  - 卖出(x→y)使用 discriminator `33e685a4017f83ad`
  - 买入时 max_sol_cost = amount_in * 2 (预留滑点空间)
  - 支持 coin_creator_vault_ata 创作者费用

#### (4) Raydium CLMM (type=3) — `raydium_clmm_market.rs`
- **Market类型**: `NormalCLMM`
- **消耗账户数**: 12个
- **核心特点**:
  - 最复杂的市场实现 (~910行代码)
  - Uniswap V3 风格的 tick-based 集中流动性
  - 使用 Q64.64 定点数表示 sqrt_price
  - 包含完整的 tick 数学: `get_sqrt_price_at_tick_x64` (查表法，19个magic number)
  - tick_array 查找、bitmap 解析、limit_in 计算
  - 支持 extension bitmap (大范围tick溢出场景)
  - 使用 U256/U128 大数运算避免溢出
  - swap discriminator: `2b04ed0b1ac91e62`

#### (5) Raydium CPMM (type=4) — `raydium_cpmm_market.rs`
- **Market类型**: `NormalAMM`
- **消耗账户数**: 9个
- **核心特点**:
  - Raydium 新版恒定乘积池
  - fee_denominator 固定 1000000
  - 余额 = vault余额 - protocol_fee - fund_fee
  - swap discriminator: `8fbe5adac41e33de`
  - 支持 Token-2022 (分别传入 token_x_program 和 token_y_program)

#### (6) Meteora AMM (type=5) — `meteora_amm_market.rs`
- **Market类型**: `NormalAMM`
- **消耗账户数**: 13个 (最多)
- **核心特点**:
  - 含 Vault 层的动态流动性 AMM
  - 需计算 vault_withdrawable_amount (考虑 locked_profit 衰减)
  - `locked_profit_degradation` 时间衰减机制
  - 通过 LP share 计算实际可用流动性
  - swap discriminator: `f8c69e91e17587c8` (与 Meteora DLMM 相同)
  - 额外需要 vault_program 账户

---

## 四、核心套利引擎分析 (`processor.rs`)

### 4.1 执行流程

```
main_process()
  │
  ├── 1. 创建中间token的ATA账户 (create_associated_token_account)
  │
  ├── 2. 构建市场列表 (create_market_list)
  │     └── 根据 market_type[] 和 market_flag[] 参数
  │         从统一的 account_list 中按顺序消耗账户
  │         每种market消耗不同数量的账户
  │
  ├── 3. 记录交易前 base_token 余额
  │
  ├── 4. 执行套利 (process_arb)
  │     └── O(n²) 双重循环遍历所有市场对
  │         ├── 跳过无效市场 (!valid())
  │         ├── 跳过 x_mint 不同的市场对
  │         └── 对每个有效市场对调用 try_arb
  │
  ├── 5. 验证利润 (after - before >= min_profit)
  │     ├── 若利润不足 → 返回 FakeProfit 错误 (整笔交易回滚)
  │     └── 若有利润 → 抽取 10% 手续费转给 recipient
  │
  └── 6. 关闭中间token的ATA账户 (回收SOL rent)
```

### 4.2 套利数学计算 (`market_arb_calc`)

该函数实现了 **9种** 市场类型组合的最优套利金额解析计算:

| 组合 | Market1 → Market2 | 算法 |
|------|-------------------|------|
| 1 | AMM → AMM | 二次方程求解最优 dx (含正负根) |
| 2 | CMM → CMM | 价差比率计算 + 流动性限制 |
| 3 | AMM → CMM | 混合模型求解 (含 sqrt) |
| 4 | CMM → AMM | 混合模型求解 (含 sqrt) |
| 5 | CLMM → AMM | CLMM流动性 + AMM恒积公式联立 |
| 6 | CLMM → CMM | CLMM流动性 + CMM价格联立 |
| 7 | CLMM → CLMM | 双CLMM流动性联立 |
| 8 | AMM → CLMM | AMM恒积 + CLMM流动性联立 |
| 9 | CMM → CLMM | CMM价格 + CLMM流动性联立 |

**核心数学思路**: 对每种组合，利用拉格朗日优化或代数求解，找到使 `out_y - in_y` 最大化的输入金额 `in_y`。
- 所有公式都推导出解析解 (closed-form solution)，不使用迭代
- 每个公式都有正根和负根两个解，优先取正解
- 所有计算使用 f64 浮点数 (链上计算！需注意精度)

### 4.3 try_arb 执行逻辑

```rust
fn try_arb(market1, market2, max_in, min_profit):
  1. 调用 market_arb_calc 计算最优 (in_y, dx, out_y)
  2. 检查: in_y > 1 && dx > 1 && out_y > 1 && out_y > in_y + min_profit
  3. 限制: in_y = min(in_y, max_in)
  4. 执行 market1.swap(y2x=true, in_y)  // base → intermediate
  5. 读取实际获得的 intermediate token 数量
  6. 执行 market2.swap(y2x=false, in_x)  // intermediate → base
```

---

## 五、账户模型设计

### 5.1 统一账户数组设计

合约使用了一种巧妙的**统一账户数组**设计:

- `CommonAccountsInfo64`: 9个固定账户 + 55个动态账户 (account_0 ~ account_54)
- `CommonAccountsInfo32`: 9个固定账户 + 29个动态账户 (account_0 ~ account_28)

所有动态账户为 `Option<UncheckedAccount>`, 通过 `AccountsIter` 迭代器按需消耗:

```
例: market_type = [0, 1]  (Meteora DLMM + Raydium AMM)
account_0 ~ account_9:  被 Meteora DLMM 消耗 (10个)
account_10 ~ account_14: 被 Raydium AMM 消耗 (5个)
其余为 None
```

### 5.2 各市场账户消耗数量

| DEX | 消耗账户数 |
|-----|-----------|
| Meteora DLMM | 10 |
| Raydium AMM | 5 |
| PumpSwap | 11 |
| Raydium CLMM | 12 |
| Raydium CPMM | 9 |
| Meteora AMM | 13 |

### 5.3 固定账户

| 账户 | 用途 |
|------|------|
| `user` | 签名者 (Signer) |
| `user_token_base` | 用户base token ATA (通常为WSOL) |
| `token_base_mint` | base token mint |
| `token_program` | SPL Token程序 |
| `sys_program` | System程序 |
| `token_pair_0_user_token_account_x` | 中间token用户ATA |
| `token_pair_0_mint_x` | 中间token mint |
| `recipient` | 手续费接收地址 (硬编码) |
| `associated_token_program` | ATA程序 |

---

## 六、market_flag 编码设计

`market_flag` 每个字节编码两个信息:
- **bit 7 (最高位)**: `reverse` 标志 (1=反向交易对)
- **bit 0~6**: `mint_x_index` (中间token索引，目前只支持0)

```
market_flag = 0b1000_0000  → reverse=true, mint_x_index=0
market_flag = 0b0000_0000  → reverse=false, mint_x_index=0
```

### MockReverseMarketPool 机制

当 `reverse=true` 时，用 `MockReverseMarketPool` 包装原始market:
- 交换 x 和 y 的所有属性
- swap 时翻转 y2x 方向
- sqrt_price 取倒数
- 实现**统一的套利方向**: 始终是 base → intermediate → base

---

## 七、关键技术细节

### 7.1 链上数据解析

所有市场都通过**原始字节偏移**解析链上账户数据，不使用 IDL 反序列化:

```rust
// 例: Raydium AMM 解析 fee
fee_numerator_buffer.copy_from_slice(&amm_data[128 + 48..128 + 48 + 8]);
```

优势: 节省计算单元 (CU)，避免引入外部 crate 依赖
风险: 如果目标DEX更新账户布局，偏移量需要同步更新

### 7.2 CPI Swap 调用

所有swap通过 `invoke()` 进行 CPI 调用，使用硬编码的指令 discriminator:

| DEX | Discriminator | 备注 |
|-----|--------------|------|
| Meteora DLMM | `f8c69e91e17587c8` | swap instruction |
| Raydium AMM | `09` (opcode) | swap_base_in |
| PumpSwap Buy | `66063d1201daebea` | buy |
| PumpSwap Sell | `33e685a4017f83ad` | sell |
| Raydium CLMM | `2b04ed0b1ac91e62` | swap |
| Raydium CPMM | `8fbe5adac41e33de` | swap_base_input |
| Meteora AMM | `f8c69e91e17587c8` | swap |

### 7.3 利润保护机制

1. **min_profit 参数**: 最小预期利润，套利计算时直接过滤
2. **FakeProfit 检查**: 交易后验证实际余额变化 >= min_profit
3. **原子性**: 如果利润不足，整笔交易回滚 (Solana事务原子性)

### 7.4 手续费模型

```rust
let profit = after_base_amount - before_base_amount;
let fee = profit.div_ceil(10);  // 10% 利润抽成，向上取整
// 通过 transfer_checked 转给 recipient (decimals=9, 假设base为SOL/WSOL)
```

---

## 八、风险与潜在问题分析

### 8.1 安全相关

| 风险 | 说明 | 严重度 |
|------|------|--------|
| **UncheckedAccount 使用** | 所有动态账户均为 UncheckedAccount，不验证 owner/program | 高 |
| **浮点精度** | 链上使用 f64 进行套利计算，可能导致精度损失 | 中 |
| **缺少 Signer 验证** | 部分CPI中 user 作为 signer 传递但未校验 authority | 中 |
| **硬编码 decimals=9** | `transfer_checked` 固定 decimals=9，仅适用于 SOL | 低 |
| **min_amount_out = 0** | 所有swap的最小输出设为0，依赖最终利润检查 | 低 |

### 8.2 功能局限

| 局限 | 说明 |
|------|------|
| **仅支持2-hop路径** | 只支持 A→B→A 的两跳套利，不支持三角或多跳 |
| **单一中间token** | 当前只支持1个中间token对 (mint_x_index只有0) |
| **O(n²)遍历** | 市场对匹配是暴力遍历，市场数多时CU消耗大 |
| **单tick/单bin** | CLMM和DLMM只读取当前流动性，不跨tick/bin |
| **无闪电贷** | 使用自有资金，不利用闪电贷放大收益 |

### 8.3 性能考量

| 项目 | 说明 |
|------|------|
| **CU消耗** | 代码中有 `sol_log_compute_units()` 用于监控 |
| **64账户版本** | 提供64账户版本支持更多市场组合 |
| **ATA创建/关闭** | 每笔交易创建并关闭中间token ATA (回收rent) |
| **debug-out feature** | 可选的预估输出计算 (仅debug模式开启) |

---

## 九、Off-chain 配合推测

根据合约设计，off-chain Bot 需要完成:

1. **监听**: 监控支持的6种DEX的池子状态变化
2. **计算**: 使用相同的 `market_arb_calc` 数学公式计算最优路径
3. **编排**: 按照 market_type[] 顺序排列所需账户，填入 account_list
4. **设置**: 计算 max_in 和 min_profit 参数
5. **发送**: 构建并发送交易 (可能配合 Jito bundle 抢先)
6. **模拟**: 可能先 simulate 验证盈利性后再提交

---

## 十、总结

这是一个**设计成熟的 Solana 链上套利合约**，具有以下特点:

**优势**:
- 统一的市场抽象层，新增DEX只需实现 `BaseMarketPool` + `CreateMarket` trait
- 解析解套利计算，无迭代，CU效率高
- 支持6种主流 Solana DEX，覆盖 AMM/CMM/CLMM 三种模型
- MockReverse 设计使正向/反向交易统一处理
- 完整的利润验证和原子回滚保护
- AccountsIter 设计灵活支持不同market组合

**待改进**:
- 可考虑支持多跳路径 (A→B→C→A)
- 浮点计算可改用定点数提高精度
- CLMM/DLMM 可支持跨 tick/bin 套利
- 可引入更多 DEX (如 Orca Whirlpool、Jupiter等)
- 账户验证可以加强 (owner check)

---

## 十一、与 solana-onchain-arbitrage-bot (Repo B) 对比分析

### 基本信息对比

| 维度 | 本仓库 (Repo A: arb_touyi) | Repo B (second_anchor / zooey_go) |
|------|---------------------------|-----------------------------------|
| **合约名** | `arb_touyi` | `second_anchor` / `zooey_go` |
| **Program ID** | `DxeQQ7PQ94j26ism5ivTqNHAkteFNmgRpqYx7XQFqs9Z` | `11111111111111111111111111111111` (未部署) |
| **Anchor 版本** | 0.30.1 | 0.31.1 (更新) |
| **代码量** | ~6,346 行 Rust | ~22,662 行 Rust (3.6倍) |
| **DEX 数量** | 6 种 | 7 种 (多 Orca Whirlpool + Meteora DAMM V2) |
| **路径** | 仅 2-hop (A→B→A) | 2-hop + **3-hop** (A→B→C→A) |
| **利润抽成** | 10% 硬编码 | **无抽成** |
| **部署状态** | 已部署 mainnet | 未部署 (program ID 全1) |

### 架构设计对比

| 维度 | Repo A (arb_touyi) | Repo B (zooey_go) |
|------|-------------------|-------------------|
| **账户传入方式** | 固定 `Option<UncheckedAccount>` 数组 (32/64个) | `remaining_accounts` 动态解析 |
| **市场抽象** | `BaseMarketPool` trait + `CreateMarket` trait | `ParsedPoolState` enum + 独立 `dex/` + `swap/` 模块 |
| **方向处理** | `MockReverseMarketPool` 装饰器模式 | 每个 DEX swap 函数内部 `is_buy` 参数控制 |
| **数学精度** | **f64 浮点数** (链上！) | **Q64.64 定点数** (u128) |
| **最优金额计算** | 9种组合的解析解 (closed-form) | **黄金分割法** 数值搜索 (`find_optimal_wsol_amount_golden_section`) |
| **价格比较** | 隐式 (在 arb_calc 中) | 显式 Q64.64 价格比率 + 利润因子计算 |
| **DEX识别** | `market_type[]` 字节数组 (off-chain编码) | 自动识别 program_id → pool_type |
| **模拟模式** | 无 (debug-out feature flag) | `is_simulate` 参数，支持链上模拟不执行 |
| **Token-2022** | 部分支持 (仅 Raydium CPMM) | **全面支持** (transfer fee 计算, token program 识别) |
| **内存管理** | 无特殊处理 | 显式 Box 堆分配 + 用后即 None 释放 |

### 支持的 DEX 对比

| DEX | Repo A | Repo B |
|-----|--------|--------|
| Raydium AMM (Legacy) | ✅ | ✅ |
| Raydium CPMM | ✅ | ✅ |
| Raydium CLMM | ✅ | ✅ |
| Meteora DLMM | ✅ | ✅ |
| Meteora AMM (Vault) | ✅ | ❌ |
| **Meteora DAMM V2** | ❌ | ✅ |
| PumpSwap | ✅ | ✅ |
| **Orca Whirlpool** | ❌ | ✅ |

### 各维度优劣判定

#### 1. 数学精度: Repo B 胜

| | Repo A | Repo B |
|--|--------|--------|
| 表示 | f64 浮点数 | Q64.64 定点数 (u128) |
| 精度 | ~15位有效数字，有舍入误差 | 精确整数运算，无浮点误差 |
| 溢出保护 | 依赖 f64 范围 | checked_sub / safe_mul_div_cast |

Repo A 在链上使用 f64 是**高风险选择** — 浮点运算在不同硬件上可能产生不同结果，且舍入累积可能导致套利金额偏差。Repo B 全程使用 Q64.64 定点数是行业最佳实践。

#### 2. 最优金额计算: 各有优劣

| | Repo A | Repo B |
|--|--------|--------|
| 方法 | 解析解 (closed-form) | 黄金分割法搜索 |
| CU 消耗 | 低 (单次计算) | 较高 (多次迭代) |
| 准确度 | 受 f64 精度限制 | 更精确 (实际模拟 swap 过程) |
| 覆盖度 | 9种市场组合都有公式 | 通用 (任意 DEX 组合自动适配) |

Repo A 的解析解在 CU 效率上更优，但因使用 f64 实际精度可能反而不如 Repo B 的数值搜索。Repo B 的黄金分割法更**通用**——增加新 DEX 不需要推导新公式。

#### 3. 路径支持: Repo B 胜

Repo B 支持 **3-hop 路径** (WSOL→Token1→Token2→WSOL)，这意味着能捕获三角套利机会。这是显著优势——很多套利机会只存在于三角路径中，2-hop 找不到。

#### 4. 账户模型: Repo B 胜

| | Repo A | Repo B |
|--|--------|--------|
| 方式 | 固定 Option 数组 | remaining_accounts 动态 |
| 灵活性 | 受 32/64 上限限制 | 无硬上限 |
| DEX 识别 | off-chain 编码 market_type | **自动识别** program_id |
| 扩展性 | 新增 DEX 需改 input_model | 新增 DEX 只需加 parse 逻辑 |

Repo B 通过 `remaining_accounts` + 自动 program_id 识别的方式更灵活优雅。Repo A 的固定数组方式虽然更简单直接，但扩展性差。

#### 5. 代码工程质量: Repo B 胜

| | Repo A | Repo B |
|--|--------|--------|
| 关注点分离 | market 中混合 dex 解析 + swap | dex/ (解析) 与 swap/ (执行) 完全分离 |
| 内存管理 | 无显式管理 | Box 堆分配 + 及时释放 + 栈优化注释 |
| 错误类型 | 5 种 | 25+ 种，覆盖更全 |
| Token-2022 | 部分 | 完整 (transfer fee 计算) |
| 中文注释 | 少量 | 丰富的中文注释和文档 |

#### 6. 实战部署: Repo A 胜

| | Repo A | Repo B |
|--|--------|--------|
| Program ID | 真实已部署 | 全 1 (占位) |
| 抽成机制 | 有 (可运营) | 无 |
| 稳定性 | 经过 mainnet 实战 | 未知 |
| 代码精简度 | 6K 行，简洁 | 22K 行，有冗余 |

Repo A 是**经过 mainnet 验证的生产代码**，这一点非常重要。Repo B 虽然功能更丰富，但 Program ID 为全 1 说明从未部署。

#### 7. 抽象设计: Repo A 更优雅

Repo A 的 `BaseMarketPool` trait + `MockReverseMarketPool` 装饰器是非常优雅的 OOP 设计:
- 新增 DEX 只需实现 trait
- 方向翻转通过装饰器自动处理
- 套利引擎完全与 DEX 解耦

Repo B 使用 enum 匹配，每增加一个 DEX 需要在 comparison.rs / optimalamt.rs / swap.rs 等多处添加 match arm，耦合度更高。

### 综合评分

| 维度 | Repo A | Repo B | 说明 |
|------|--------|--------|------|
| 数学精度 | 6/10 | **9/10** | Q64.64 vs f64 |
| 路径丰富度 | 5/10 | **9/10** | 2-hop vs 2+3-hop |
| DEX 覆盖度 | 7/10 | **8/10** | 6 vs 7，且 Whirlpool 重要 |
| 抽象设计 | **9/10** | 7/10 | Trait 优于 enum 大量 match |
| 代码精简度 | **9/10** | 6/10 | 6K vs 22K，信息密度高 |
| CU 效率 | **8/10** | 6/10 | 解析解 vs 迭代搜索 |
| 扩展性 | 6/10 | **8/10** | remaining_accounts 更灵活 |
| Token-2022 | 4/10 | **9/10** | 全面 vs 部分 |
| 实战验证 | **10/10** | 3/10 | 已部署 vs 未部署 |
| 内存安全 | 5/10 | **8/10** | 无管理 vs 显式 Box |
| **总计** | **69/100** | **73/100** | |

### 结论

**Repo B (zooey_go) 在技术完整度上更优秀**：
- Q64.64 定点数精度更高
- 3-hop 路径覆盖更多套利机会
- 7 种 DEX (含 Orca Whirlpool) 覆盖更广
- Token-2022 全面支持
- 内存管理更专业

**Repo A (arb_touyi) 在工程效率和实战上更优秀**：
- 代码量仅 1/3.6，但实现了核心功能
- 解析解套利计算，CU 效率更高
- Trait 抽象设计更优雅
- **已在 mainnet 实战验证**——这是最重要的

**如果要做一个最优方案**，应该取二者之长：
1. 用 Repo A 的 trait 抽象架构
2. 用 Repo B 的 Q64.64 定点数精度
3. 用 Repo B 的 remaining_accounts 灵活账户传入
4. 用 Repo B 的 3-hop 路径支持
5. 保留 Repo A 的解析解 (CU 优势) 但用定点数重写
6. 加入 Repo B 的 Orca Whirlpool 和 Token-2022 全面支持
