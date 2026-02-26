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
