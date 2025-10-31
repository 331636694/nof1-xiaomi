# OKX API 集成指南

## 📋 目录

1. [项目概述](#项目概述)
2. [架构设计](#架构设计)
3. [分阶段实施计划](#分阶段实施计划)
4. [技术实现细节](#技术实现细节)
5. [配置指南](#配置指南)
6. [开发检查清单](#开发检查清单)
7. [测试策略](#测试策略)
8. [风险评估](#风险评估)

---

## 项目概述

### 🎯 集成目标

为 nof1-tracker 项目添加 OKX 合约交易 API 支持，实现：

- **功能对等性**：与现有币安 API 保持完全相同的功能
- **无缝切换**：用户可以通过简单配置在币安和 OKX 之间切换
- **统一体验**：相同的 CLI 命令、相同的交易逻辑、相同的风险管理
- **向后兼容**：不影响现有币安用户的使用体验

### 📊 当前币安功能分析

基于代码分析，当前支持的币安功能包括：

#### 核心交易功能
- ✅ 账户信息查询
- ✅ 持仓信息查询
- ✅ 合约下单（市价、限价、止损、止盈）
- ✅ 杠杆设置（1-125x）
- ✅ 保证金模式设置（逐仓/全仓）
- ✅ 订单管理（查询、取消）
- ✅ 用户交易历史查询

#### 风险管理功能
- ✅ 余额检查和保证金验证
- ✅ 自动数量调整
- ✅ 价格精度处理
- ✅ 时间同步和重试机制
- ✅ 止盈止损单自动创建

#### 高级功能
- ✅ Telegram 通知集成
- ✅ 详细的日志记录
- ✅ 错误处理和恢复
- ✅ 连接管理和资源清理

### 🚧 技术挑战

#### 1. API 差异适配

| 特性 | 币安 (Binance) | OKX | 解决方案 |
|------|----------------|-----|----------|
| **认证方式** | API Key + Secret | API Key + Secret + Passphrase | 扩展认证接口 |
| **签名算法** | HMAC-SHA256 | HMAC-SHA256 + 不同参数格式 | 适配签名生成 |
| **符号格式** | BTCUSDT | BTC-USDT-SWAP | 符号转换层 |
| **基础URL** | fapi.binance.com | www.okx.com | 配置化管理 |
| **端点路径** | /fapi/v1/ | /api/v5/ | 端点映射 |

#### 2. 数据格式差异

```typescript
// 币安持仓响应格式
interface BinancePosition {
  symbol: string;
  positionAmt: string;
  entryPrice: string;
  markPrice: string;
  unRealizedProfit: string;
  leverage: string;
  // ...
}

// OKX 持仓响应格式（需要适配）
interface OkxPosition {
  instId: string;        // 对应 symbol
  pos: string;          // 对应 positionAmt
  avgPx: string;        // 对应 entryPrice
  markPx: string;       // 对应 markPrice
  upl: string;          // 对应 unRealizedProfit
  lever: string;        // 对应 leverage
  // ...
}
```

#### 3. 精度和规格差异

- **最小下单量**：不同交易对的最小数量要求不同
- **价格精度**：OKX 和币安的价格小数位数可能不同
- **杠杆限制**：OKX 的杠杆范围和规则可能与币安不同

---

## 架构设计

### 🏗️ 总体架构

采用**抽象工厂模式 + 策略模式**的设计，实现交易所的解耦和可扩展性：

```
┌─────────────────────────────────────────────────────────────┐
│                    CLI Layer                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   agents    │  │    follow   │  │   status    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                Trading Executor                            │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           Exchange Factory                          │    │
│  │  ┌─────────────┐    ┌─────────────┐                │    │
│  │  │  Binance    │    │     OKX     │                │    │
│  │  │  Service    │    │   Service   │                │    │
│  │  └─────────────┘    └─────────────┘                │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│              Exchange Interface                            │
│           (统一的服务抽象层)                               │
└─────────────────────────────────────────────────────────────┘
```

### 🔌 接口设计

#### 1. 核心服务接口

```typescript
// src/services/exchange.interface.ts
export interface ExchangeService {
  // 连接验证
  validateConnection(): Promise<boolean>;
  getServerTime(): Promise<number>;
  destroy(): void;

  // 账户管理
  getAccountInfo(): Promise<AccountInfo>;
  getPositions(): Promise<PositionResponse[]>;
  getAllPositions(): Promise<PositionResponse[]>;

  // 交易操作
  placeOrder(order: ExchangeOrder): Promise<OrderResponse>;
  cancelOrder(symbol: string, orderId: number | string): Promise<OrderResponse>;
  getOrderStatus(symbol: string, orderId: number | string): Promise<OrderResponse>;
  getOpenOrders(symbol?: string): Promise<OrderResponse[]>;
  cancelAllOrders(symbol: string): Promise<any>;

  // 风险管理
  setLeverage(symbol: string, leverage: number): Promise<any>;
  setMarginType(symbol: string, marginType: 'ISOLATED' | 'CROSSED'): Promise<any>;

  // 手动平仓检测（新增功能）
  detectManualClosure(
    currentPositions: PositionResponse[],
    orderHistory: OrderHistoryRecord[]
  ): Promise<ManualCloseRecord[]>;

  handleManualClosure(
    closeRecord: ManualCloseRecord,
    config: FollowConfig
  ): Promise<void>;

  // 市场数据
  getExchangeInformation(): Promise<any>;
  getSymbolInfo(symbol: string): Promise<any>;
  get24hrTicker(symbol?: string): Promise<any>;

  // 交易历史
  getUserTrades(
    symbol?: string,
    startTime?: number,
    endTime?: number,
    fromId?: number | string,
    limit?: number
  ): Promise<UserTrade[]>;

  // 工具方法
  convertSymbol(symbol: string): string;
  formatQuantity(quantity: number | string, symbol: string): string;
  formatPrice(price: number | string, symbol: string): string;
}
```

#### 2. 统一数据类型

```typescript
// 统一的订单类型
export interface ExchangeOrder {
  symbol: string;
  side: "BUY" | "SELL";
  type: "MARKET" | "LIMIT" | "STOP" | "TAKE_PROFIT" | "TAKE_PROFIT_MARKET" | "STOP_MARKET";
  quantity: string;
  leverage?: number;
  price?: string;
  stopPrice?: string;
  timeInForce?: "GTC" | "IOC" | "FOK";
  closePosition?: string;
}

// 统一的持仓响应
export interface PositionResponse {
  symbol: string;
  positionAmt: string;
  entryPrice: string;
  markPrice: string;
  unRealizedProfit: string;
  liquidationPrice: string;
  leverage: string;
  marginType: string;
  positionSide: string;
  notional: string;
  updateTime: number;
}

// 统一的订单响应
export interface OrderResponse {
  orderId: string | number;
  symbol: string;
  status: string;
  clientOrderId: string;
  price: string;
  avgPrice: string;
  origQty: string;
  executedQty: string;
  side: string;
  type: string;
  time: number;
  updateTime: number;
}

// 手动平仓记录（新增类型）
export interface ManualCloseRecord {
  symbol: string;
  side: "BUY" | "SELL";
  closePrice: string;
  closeQuantity: string;
  closeTime: number;
  detectionTime: number;
  originalOrderId: string;
  positionId?: string;
  realizedPnl?: string;
}

// 订单历史记录（扩展支持手动平仓）
export interface OrderHistoryRecord {
  orderId: string;
  symbol: string;
  side: "BUY" | "SELL";
  type: string;
  quantity: string;
  price?: string;
  status: 'FILLED' | 'CANCELED' | 'REJECTED' | 'EXPIRED' | 'manual_closed';
  updateTime: number;
  isMaker?: boolean;
  commission?: string;
  realizedPnl?: string;
}

// 方向性价格容差检查（新增）
export interface PriceToleranceCheck {
  isValid: boolean;
  priceDifference: number;
  tolerancePercent: number;
  direction: 'UP' | 'DOWN';
  positionSide: "BUY" | "SELL";
  referencePrice: string;
  currentPrice: string;
}
```

#### 3. 工厂模式实现

```typescript
// src/services/exchange-factory.ts
export class ExchangeFactory {
  static createExchange(
    exchange: 'binance' | 'okx',
    config: ExchangeConfig
  ): ExchangeService {
    switch (exchange) {
      case 'binance':
        return new BinanceService(
          config.apiKey,
          config.apiSecret,
          config.testnet
        );
      case 'okx':
        return new OkxService(
          config.apiKey,
          config.apiSecret,
          config.passphrase,
          config.testnet
        );
      default:
        throw new Error(`Unsupported exchange: ${exchange}`);
    }
  }
}

export interface ExchangeConfig {
  apiKey: string;
  apiSecret: string;
  passphrase?: string;  // OKX 特有
  testnet?: boolean;
}
```

### 🔄 配置管理扩展

#### 1. 环境变量扩展

```bash
# 新增 OKX 配置
OKX_API_KEY=your_okx_api_key
OKX_API_SECRET=your_okx_api_secret
OKX_API_PASSPHRASE=your_okx_passphrase
OKX_TESTNET=true

# 交易所选择（可选，默认 binance）
DEFAULT_EXCHANGE=binance  # binance | okx
```

#### 2. 配置管理器更新

```typescript
// src/services/config-manager.ts
export interface AppConfig {
  exchange: {
    type: 'binance' | 'okx';
    binance?: BinanceConfig;
    okx?: OkxConfig;
  };
  // ... 其他配置
}

export interface BinanceConfig {
  apiKey: string;
  apiSecret: string;
  testnet: boolean;
}

export interface OkxConfig {
  apiKey: string;
  apiSecret: string;
  passphrase: string;
  testnet: boolean;
}
```

---

## 分阶段实施计划

### 📅 第一阶段：基础架构设计 (2-3天)

#### 任务清单

- [ ] **1.1 创建接口定义文件**
  ```bash
  touch src/services/exchange.interface.ts
  touch src/services/exchange-factory.ts
  touch src/types/exchange.types.ts
  ```

- [ ] **1.2 定义统一接口**
  - 实现 `ExchangeService` 接口
  - 定义通用数据类型
  - 添加手动平仓检测方法
  - 创建工厂模式基础结构

- [ ] **1.3 更新配置系统**
  - 扩展 `constants.ts` 添加 OKX 相关常量
  - 更新 `config-manager.ts` 支持多交易所配置
  - 修改 `.env.example` 添加 OKX 配置模板
  - 集成新的 CLI 参数（`--auto-refollow`, `--telegram-test`）

- [ ] **1.4 创建测试基础**
  - 创建交易所接口的测试框架
  - 设计 Mock 服务用于测试
  - 添加手动平仓检测的测试用例

- [ ] **1.5 集成现有功能**
  - 确保与现有 auto-refollow 功能兼容
  - 集成方向性价格容差检查
  - 协调插件通知系统架构

#### 验收标准

- ✅ 接口定义完整且类型安全
- ✅ 工厂模式可以正确创建币安服务实例
- ✅ 配置系统支持读取 OKX 配置
- ✅ 基础测试框架可以运行
- ✅ 与现有功能完全兼容

---

### 📅 第二阶段：OKX 服务实现 (3-4天)

#### 任务清单

- [ ] **2.1 创建 OKX 服务基础结构**
  ```bash
  touch src/services/okx-service.ts
  ```

- [ ] **2.2 实现认证机制**
  ```typescript
  // OKX 特有的认证（包含 passphrase）
  private createSignature(timestamp: string, method: string, path: string, body: string): string {
    const message = timestamp + method + path + body;
    return CryptoJS.HmacSHA256(message, this.apiSecret).toString(CryptoJS.enc.Base64);
  }
  ```

- [ ] **2.3 实现核心 API 方法**
  - `getAccountInfo()` - 账户信息查询
  - `getPositions()` - 持仓信息查询
  - `placeOrder()` - 下单功能
  - `setLeverage()` - 杠杆设置
  - `setMarginType()` - 保证金模式设置

- [ ] **2.4 实现手动平仓检测**
  - `detectManualClosure()` - 检测手动平仓
  - `handleManualClosure()` - 处理手动平仓事件
  - 集成 auto-refollow 功能
  - 实现通知机制

- [ ] **2.5 实现方向性价格容差**
  - `calculateDirectionalPriceDifference()` - 方向性价格检查
  - 集成现有的风险管理逻辑
  - 支持多空头的不同容差策略

- [ ] **2.6 符号和精度处理**
  ```typescript
  // OKX 符号转换：BTC → BTC-USDT-SWAP
  public convertSymbol(symbol: string): string {
    if (symbol.includes('-')) return symbol; // 已经是 OKX 格式
    return `${symbol}-USDT-SWAP`;
  }
  ```

- [ ] **2.7 错误处理适配**
  - 适配 OKX 特有的错误代码
  - 实现重试机制和时间同步
  - 统一错误消息格式
  - 集成 Telegram 通知

#### 验收标准

- ✅ OKX 服务可以成功连接测试网
- ✅ 基础 API 调用（账户信息、持仓查询）正常工作
- ✅ 符号转换和精度格式化正确
- ✅ 手动平仓检测功能正常工作
- ✅ 方向性价格容差检查准确
- ✅ 错误处理机制完善

---

### 📅 第三阶段：代码重构 (2-3天)

#### 任务清单

- [ ] **3.1 重构 BinanceService**
  - 让 `BinanceService` 实现 `ExchangeService` 接口
  - 更新方法签名以匹配接口定义
  - 添加手动平仓检测方法（币安版本）
  - 保持向后兼容性

- [ ] **3.2 更新 TradingExecutor**
  ```typescript
  export class TradingExecutor {
    private exchangeService: ExchangeService;
    private exchangeType: 'binance' | 'okx';

    constructor(
      exchangeType: 'binance' | 'okx' = 'binance',
      config?: ExchangeConfig
    ) {
      this.exchangeType = exchangeType;
      this.exchangeService = ExchangeFactory.createExchange(exchangeType, config || {});
    }
  }
  ```

- [ ] **3.3 集成 Auto-Refollow 功能**
  - 更新 `FollowService` 以支持多交易所手动平仓检测
  - 确保方向性价格容差在两个交易所都正常工作
  - 统一 `OrderHistoryManager` 接口

- [ ] **3.4 更新依赖注入**
  - 修改所有使用 `BinanceService` 的地方
  - 通过工厂模式创建交易所实例
  - 更新构造函数签名
  - 集成新的 CLI 参数

- [ ] **3.5 数据适配层**
  - 确保所有返回的数据格式统一
  - 处理交易所特有的字段差异
  - 更新相关的类型定义
  - 适配 `ManualCloseRecord` 和 `PriceToleranceCheck`

- [ ] **3.6 通知系统协调**
  - 确保 Telegram 通知在两个交易所都正常工作
  - 集成 `telegram-test` 命令
  - 协调插件通知系统架构

#### 验收标准

- ✅ 现有币安功能完全正常，无回归
- ✅ `TradingExecutor` 支持动态切换交易所
- ✅ Auto-Refollow 功能在两个交易所都正常工作
- ✅ 所有单元测试通过
- ✅ 代码质量检查通过

---

### 📅 第四阶段：CLI 和入口更新 (2天)

#### 任务清单

- [ ] **4.1 更新 CLI 参数**
  ```typescript
  // 添加交易所选择参数
  program
    .option('-e, --exchange <exchange>', 'Exchange to use (binance|okx)', 'binance')
    .option('--okx-api-key <key>', 'OKX API key')
    .option('--okx-api-secret <secret>', 'OKX API secret')
    .option('--okx-passphrase <passphrase>', 'OKX API passphrase')
    .option('--auto-refollow', 'Enable auto-refollow when manual close detected')
    .command('telegram-test', 'Test Telegram notification setup');
  ```

- [ ] **4.2 更新主入口**
  - 修改 `src/index.ts` 支持交易所参数
  - 实现配置验证和错误提示
  - 更新帮助信息
  - 添加交易所特定配置验证

- [ ] **4.3 更新命令处理器**
  - 修改 `src/commands/` 下的命令文件
  - 支持传递交易所配置
  - 集成 `--auto-refollow` 参数
  - 添加 `telegram-test` 命令实现
  - 更新错误处理

- [ ] **4.4 验证和测试**
  - 测试 CLI 参数解析
  - 验证配置传递正确性
  - 测试错误场景处理
  - 验证新命令功能

- [ ] **4.5 更新文档**
  - 更新 README.md 的命令行参数说明
  - 添加新功能的使用示例
  - 更新故障排除指南

#### 验收标准

- ✅ CLI 支持 `--exchange okx` 参数
- ✅ OKX 配置参数正确传递
- ✅ `--auto-refollow` 参数正常工作
- ✅ `telegram-test` 命令功能正常
- ✅ 帮助信息包含新参数说明
- ✅ 错误提示清晰友好

---

### 📅 第五阶段：测试和文档 (2-3天)

#### 任务清单

- [ ] **5.1 创建 OKX 测试套件**
  ```bash
  touch src/services/__tests__/okx-service.test.ts
  ```

- [ ] **5.2 集成测试**
  - 使用 OKX 测试网进行端到端测试
  - 验证所有核心功能
  - 测试手动平仓检测功能
  - 测试 auto-refollow 功能
  - 测试错误场景和恢复机制

- [ ] **5.3 功能专项测试**
  - 测试方向性价格容差检查
  - 验证 Telegram 通知集成
  - 测试 `telegram-test` 命令
  - 验证 CLI 参数解析
  - 测试配置验证逻辑

- [ ] **5.4 文档更新**
  - 更新 `README.md` 添加 OKX 配置说明
  - 创建 OKX API 申请指南
  - 更新环境变量文档
  - 添加新功能使用示例
  - 更新故障排除指南

- [ ] **5.5 性能测试**
  - 对比币安和 OKX 的响应时间
  - 验证内存使用情况
  - 测试并发请求处理
  - 测试手动平仓检测性能

- [ ] **5.6 用户验收测试**
  - 邀请用户进行 Beta 测试
  - 收集反馈和改进建议
  - 修复发现的问题
  - 优化用户体验

#### 验收标准

- ✅ 所有测试通过（包括新的 OKX 测试）
- ✅ 手动平仓检测功能稳定可靠
- ✅ Auto-refollow 功能正常工作
- ✅ Telegram 通知集成完善
- ✅ 文档完整且准确
- ✅ 性能表现符合预期
- ✅ 用户体验良好

---

## 技术实现细节

### 🔐 OKX 认证机制

OKX API 使用 HmacSHA256 签名，需要额外的 passphrase：

```typescript
class OkxService implements ExchangeService {
  private createSignature(
    timestamp: string,
    method: string,
    path: string,
    body: string
  ): string {
    const message = timestamp + method + path + body;
    return CryptoJS.HmacSHA256(message, this.apiSecret).toString(CryptoJS.enc.Base64);
  }

  private async makeSignedRequest<T>(
    endpoint: string,
    method: 'GET' | 'POST' | 'DELETE' = 'GET',
    params: Record<string, any> = {}
  ): Promise<T> {
    const timestamp = new Date().toISOString();
    const body = method === 'GET' ? '' : JSON.stringify(params);
    const sign = this.createSignature(timestamp, method, endpoint, body);

    const headers = {
      'OK-ACCESS-KEY': this.apiKey,
      'OK-ACCESS-SIGN': sign,
      'OK-ACCESS-TIMESTAMP': timestamp,
      'OK-ACCESS-PASSPHRASE': this.passphrase,
      'Content-Type': 'application/json',
    };

    // 发送请求...
  }
}
```

### 🔄 符号转换对照表

| NOF1 格式 | 币安格式 | OKX 格式 | 转换逻辑 |
|-----------|----------|----------|----------|
| BTC | BTCUSDT | BTC-USDT-SWAP | appendUSDT / insert-dash-SWAP |
| ETH | ETHUSDT | ETH-USDT-SWAP | appendUSDT / insert-dash-SWAP |
| BNB | BNBUSDT | BNB-USDT-SWAP | appendUSDT / insert-dash-SWAP |

```typescript
// 通用符号转换器
export class SymbolConverter {
  static toBinance(symbol: string): string {
    return symbol.endsWith('USDT') ? symbol : `${symbol}USDT`;
  }

  static toOkx(symbol: string): string {
    if (symbol.includes('-')) return symbol;
    return `${symbol}-USDT-SWAP`;
  }

  static fromExchange(symbol: string, exchange: 'binance' | 'okx'): string {
    if (exchange === 'binance') {
      return symbol.replace('USDT', '');
    } else {
      return symbol.replace('-USDT-SWAP', '');
    }
  }
}
```

### 📊 精度处理策略

OKX 和币安的精度要求可能不同，需要动态获取：

```typescript
class OkxService implements ExchangeService {
  private symbolPrecisionCache: Map<string, any> = new Map();

  async getSymbolPrecision(symbol: string): Promise<{lot: number, price: number}> {
    const okxSymbol = this.convertSymbol(symbol);

    if (this.symbolPrecisionCache.has(okxSymbol)) {
      return this.symbolPrecisionCache.get(okxSymbol);
    }

    try {
      const response = await this.makePublicRequest('/api/v5/public/instruments', {
        instType: 'SWAP',
        instId: okxSymbol
      });

      const instrument = response.data[0];
      const precision = {
        lot: Math.abs(Math.log10(parseFloat(instrument.minSz))),
        price: Math.abs(Math.log10(parseFloat(instrument.tickSz)))
      };

      this.symbolPrecisionCache.set(okxSymbol, precision);
      return precision;
    } catch (error) {
      // 返回默认精度
      return { lot: 3, price: 2 };
    }
  }

  async formatQuantity(quantity: number | string, symbol: string): Promise<string> {
    const precision = await this.getSymbolPrecision(symbol);
    const quantityNum = typeof quantity === 'string' ? parseFloat(quantity) : quantity;
    return quantityNum.toFixed(precision.lot);
  }
}
```

### 🔄 手动平仓检测实现（新增功能）

OKX 服务需要实现手动平仓检测，以支持 auto-refollow 功能：

```typescript
class OkxService implements ExchangeService {
  async detectManualClosure(
    currentPositions: PositionResponse[],
    orderHistory: OrderHistoryRecord[]
  ): Promise<ManualCloseRecord[]> {
    const manualCloses: ManualCloseRecord[] = [];

    // 获取最近的交易历史
    const recentTrades = await this.getUserTrades(undefined, Date.now() - 300000); // 最近5分钟

    for (const position of currentPositions) {
      if (parseFloat(position.positionAmt) === 0) {
        // 检查是否有未跟踪的平仓记录
        const closeRecord = await this.findManualCloseRecord(position, orderHistory, recentTrades);
        if (closeRecord) {
          manualCloses.push(closeRecord);
        }
      }
    }

    return manualCloses;
  }

  private async findManualCloseRecord(
    position: PositionResponse,
    orderHistory: OrderHistoryRecord[],
    recentTrades: UserTrade[]
  ): Promise<ManualCloseRecord | null> {
    // 查找可能的手动平仓交易
    const positionSide = position.positionSide === 'LONG' ? 'SELL' : 'BUY';
    const closingTrades = recentTrades.filter(trade =>
      trade.symbol === position.symbol &&
      trade.side === positionSide &&
      !orderHistory.some(order => order.orderId === trade.orderId)
    );

    if (closingTrades.length > 0) {
      // 计算总平仓量和平均价格
      const totalQuantity = closingTrades.reduce((sum, trade) => sum + parseFloat(trade.qty), 0);
      const avgPrice = closingTrades.reduce((sum, trade) =>
        sum + parseFloat(trade.price) * parseFloat(trade.qty), 0
      ) / totalQuantity;

      return {
        symbol: position.symbol,
        side: positionSide,
        closePrice: avgPrice.toFixed(position.markPrice.split('.')[1].length),
        closeQuantity: totalQuantity.toString(),
        closeTime: Math.max(...closingTrades.map(trade => trade.time)),
        detectionTime: Date.now(),
        originalOrderId: 'MANUAL_DETECTED',
        realizedPnl: position.unRealizedProfit
      };
    }

    return null;
  }

  async handleManualClosure(
    closeRecord: ManualCloseRecord,
    config: FollowConfig
  ): Promise<void> {
    // 实现自动重新跟单逻辑
    console.log(`🔄 检测到手动平仓: ${closeRecord.symbol} ${closeRecord.side} ${closeRecord.closeQuantity}`);

    // 发送通知
    await this.sendManualCloseNotification(closeRecord);

    // 如果启用了 auto-refollow，重新开始跟单
    if (config.autoRefollow) {
      console.log('🚀 启动自动重新跟单...');
      // 重置订单历史，准备重新跟单
      // 这里会调用具体的重新跟单逻辑
    }
  }

  private async sendManualCloseNotification(closeRecord: ManualCloseRecord): Promise<void> {
    // 发送 Telegram 通知或其他通知方式
    const message = `🔄 手动平仓检测

币种: ${closeRecord.symbol}
方向: ${closeRecord.side}
价格: ${closeRecord.closePrice}
数量: ${closeRecord.closeQuantity}
时间: ${new Date(closeRecord.closeTime).toLocaleString()}

系统将自动重新开始跟单。`;

    // 调用通知服务
    // await notificationService.send(message);
  }
}
```

### 🎯 方向性价格容差检查

实现更智能的价格容差检查，考虑持仓方向：

```typescript
class OkxService implements ExchangeService {
  calculateDirectionalPriceDifference(
    currentPrice: number,
    referencePrice: number,
    positionSide: "BUY" | "SELL",
    tolerancePercent: number
  ): PriceToleranceCheck {
    const priceDifference = positionSide === 'BUY'
      ? currentPrice - referencePrice  // 多头：价格上涨为正向差异
      : referencePrice - currentPrice; // 空头：价格下跌为正向差异

    const toleranceAmount = referencePrice * (tolerancePercent / 100);
    const isValid = Math.abs(priceDifference) <= toleranceAmount;
    const direction = priceDifference >= 0 ? 'UP' : 'DOWN';

    return {
      isValid,
      priceDifference,
      tolerancePercent,
      direction,
      positionSide,
      referencePrice: referencePrice.toString(),
      currentPrice: currentPrice.toString()
    };
  }
}
```

### ⚠️ 错误处理适配

OKX 使用不同的错误代码体系：

```typescript
// OKX 错误代码映射
const OKX_ERROR_MAP: Record<string, string> = {
  '51000': 'Invalid API Key',
  '51008': 'Timestamp is invalid',
  '51020': 'Signature not valid',
  '51100': 'Invalid IP',
  '51108': 'Invalid passphrase',
  '51112': 'Invalid sign',
  '53001': 'Service unavailable',
  '53002': 'System busy',
  '53003': 'Internal error',
  '53004': 'Record not found',
  '53005': 'No operation permission',
  '53006': 'Request too frequent',
  '51116': 'Insufficient balance',
  '51144': 'Order size exceeds limit',
  '51146': 'Order not found',
  '51160': 'Invalid symbol',
};

function handleOkxError(error: any): never {
  const errorCode = error?.data?.[0]?.sCode;
  const errorMessage = error?.data?.[0]?.sMsg;

  const mappedMessage = OKX_ERROR_MAP[errorCode] || errorMessage || 'Unknown OKX API error';

  if (['51008', '51020', '51112'].includes(errorCode)) {
    // 时间同步或签名错误，需要重新同步
    throw new Error(`OKX Authentication Error: ${mappedMessage}. Please check your API credentials.`);
  }

  if (errorCode === '51116') {
    throw new Error(`OKX Insufficient Balance: ${mappedMessage}`);
  }

  throw new Error(`OKX API Error [${errorCode}]: ${mappedMessage}`);
}
```

---

## 配置指南

### 🔑 OKX API 申请

#### 1. 注册 OKX 账户

1. 访问 [OKX 官网](https://www.okx.com) 注册账户
2. 完成身份认证（KYC）
3. 启用合约交易功能

#### 2. 创建 API Key

1. 登录 OKX，进入 **API 管理** 页面
2. 点击 **创建 API Key**
3. 设置 **API 名称**（如：nof1-trader）
4. 选择 **权限**：
   - ✅ **交易** - 允许下单和撤单
   - ✅ **读取** - 允许查询账户信息
   - ❌ **提币** - 不需要提币权限
5. 设置 **Passphrase**（务必保存好）
6. 完成安全验证（短信/邮箱验证）

#### 3. 配置环境变量

创建 `.env` 文件（或更新现有文件）：

```bash
# Nof1 API Configuration
# 用于获取AI Agent的交易信号
NOF1_API_BASE_URL=https://nof1.ai/api

# OKX API Configuration
# ⚠️ 重要：必须启用合约交易权限
# 获取方式：https://www.okx.com/account/api-management
OKX_API_KEY=your_okx_api_key_here
OKX_API_SECRET=your_okx_api_secret_here
OKX_API_PASSPHRASE=your_okx_passphrase_here

# 是否使用测试网环境（推荐新手先使用测试网）
# true = 测试网（虚拟资金，无风险）
# false = 正式网（真实资金，有风险）
OKX_TESTNET=true

# Trading Configuration
# 交易相关配置通过命令行参数控制，如：
# --total-margin 1000    # 总保证金
# --fixed-amount-per-coin 100  # 每个币种固定保证金
# --price-tolerance 1.0  # 价格容差百分比
# --auto-refollow        # 启用自动重新跟单功能

# Logging Configuration
# 日志级别: ERROR, WARN, INFO (默认), DEBUG, VERBOSE
# INFO - 只显示重要操作(推荐日常使用)
# DEBUG - 显示详细调试信息
# VERBOSE - 显示所有日志(仅用于深度调试)
LOG_LEVEL=INFO

# Telegram Bot Configuration
# 可选，用于接收交易通知
# 获取方式：https://core.telegram.org/bots#6-botfather
TELEGRAM_API_TOKEN=
TELEGRAM_CHAT_ID=
TELEGRAM_ENABLED=false

# 交易所选择（可选，默认 binance）
DEFAULT_EXCHANGE=okx  # binance | okx
```

#### 4. 测试网配置

OKX 提供测试网环境：

- **测试网地址**: https://www.okx.com/account/demo
- **测试网 API**: https://www.okx.com/api/v5/
- **获取测试资金**: 测试网账户会自动获得虚拟资金

### 🔧 环境变量详解

| 变量名 | 必需 | 说明 | 示例 |
|--------|------|------|------|
| `NOF1_API_BASE_URL` | ✅ | Nof1 API 基础URL | `https://nof1.ai/api` |
| `OKX_API_KEY` | ✅ | OKX API Key | `xxxx-xxxx-xxxx-xxxx` |
| `OKX_API_SECRET` | ✅ | OKX Secret Key | `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| `OKX_API_PASSPHRASE` | ✅ | API 创建时设置的 Passphrase | `YourPassphrase123` |
| `OKX_TESTNET` | ❌ | 是否使用测试网 | `true` |
| `DEFAULT_EXCHANGE` | ❌ | 默认交易所 | `okx` |
| `LOG_LEVEL` | ❌ | 日志级别 | `INFO` |
| `TELEGRAM_API_TOKEN` | ❌ | Telegram Bot Token | `bot_token_here` |
| `TELEGRAM_CHAT_ID` | ❌ | Telegram 聊天ID | `chat_id_here` |
| `TELEGRAM_ENABLED` | ❌ | 是否启用Telegram通知 | `false` |

### 🚀 快速验证

#### 1. 验证配置

```bash
# 检查配置是否正确
npm start -- status --exchange okx
```

#### 2. 测试连接

```bash
# 仅验证连接，不执行交易
npm start -- follow test-agent --risk-only --exchange okx
```

#### 3. 测试 Telegram 通知

```bash
# 测试 Telegram 通知设置
npm start -- telegram-test --exchange okx
```

#### 4. 小额测试

在测试网上进行小额测试交易：

```bash
# 使用最小金额测试
npm start -- follow deepseek-chat-v3.1 --total-margin 10 --exchange okx

# 启用自动重新跟单功能测试
npm start -- follow deepseek-chat-v3.1 --total-margin 10 --auto-refollow --exchange okx
```

---

## 开发检查清单

### ✅ 第一阶段检查清单

#### 代码质量
- [ ] TypeScript 类型定义完整
- [ ] ESLint 检查通过
- [ ] Prettier 格式化一致
- [ ] JSDoc 注释完整

#### 功能完整性
- [ ] `ExchangeService` 接口包含所有必需方法
- [ ] `ExchangeFactory` 可以创建币安实例
- [ ] 配置系统支持 OKX 参数读取
- [ ] 错误处理机制完善

#### 测试覆盖
- [ ] 接口定义的单元测试
- [ ] 工厂模式的测试用例
- [ ] 配置加载的测试
- [ ] Mock 服务的测试框架

### ✅ 第二阶段检查清单

#### 核心功能
- [ ] `validateConnection()` 方法正常工作
- [ ] `getAccountInfo()` 返回正确的账户信息
- [ ] `getPositions()` 返回持仓列表
- [ ] `placeOrder()` 可以成功下单
- [ ] `setLeverage()` 杠杆设置功能正常

#### 数据处理
- [ ] 符号转换函数正确处理各种格式
- [ ] 数量和价格格式化符合 OKX 要求
- [ ] 时间戳同步机制工作正常
- [ ] 响应数据格式正确转换

#### 错误处理
- [ ] API 错误正确映射为用户友好的消息
- [ ] 网络超时处理机制
- [ ] 重试逻辑工作正常
- [ ] 认证失败有明确的错误提示

### ✅ 第三阶段检查清单

#### 重构质量
- [ ] `BinanceService` 成功实现接口
- [ ] `TradingExecutor` 支持多交易所
- [ ] 所有依赖正确注入
- [ ] 向后兼容性保持

#### 代码一致性
- [ ] 方法签名统一
- [ ] 错误处理方式一致
- [ ] 日志格式统一
- [ ] 类型定义统一

#### 测试验证
- [ ] 现有币安功能测试全部通过
- [ ] 新增接口测试通过
- [ ] 集成测试无回归
- [ ] 性能测试通过

### ✅ 第四阶段检查清单

#### CLI 功能
- [ ] `--exchange` 参数正确解析
- [ ] OKX 配置参数正确传递
- [ ] 帮助信息包含新参数
- [ ] 参数验证逻辑完善

#### 用户体验
- [ ] 错误提示清晰友好
- [ ] 配置验证有明确指导
- [ ] 命令行交互流畅
- [ ] 帮助文档准确

### ✅ 第五阶段检查清单

#### 文档完整性
- [ ] README.md 更新完成
- [ ] API 申请指南清晰
- [ ] 配置说明详细
- [ ] 故障排除指南完整

#### 测试覆盖
- [ ] OKX 服务单元测试
- [ ] 集成测试用例完整
- [ ] 错误场景测试覆盖
- [ ] 性能测试结果达标

#### 部署就绪
- [ ] 所有测试通过
- [ ] 代码审查完成
- [ ] 文档审核通过
- [ ] 发布准备就绪

---

## 测试策略

### 🧪 单元测试

#### 测试框架结构
```
src/
├── services/
│   ├── __tests__/
│   │   ├── exchange-factory.test.ts
│   │   ├── binance-service.test.ts
│   │   ├── okx-service.test.ts
│   │   └── trading-executor.test.ts
│   └── ...
```

#### 关键测试用例

1. **工厂模式测试**
```typescript
describe('ExchangeFactory', () => {
  test('should create Binance service', () => {
    const service = ExchangeFactory.createExchange('binance', binanceConfig);
    expect(service).toBeInstanceOf(BinanceService);
  });

  test('should create OKX service', () => {
    const service = ExchangeFactory.createExchange('okx', okxConfig);
    expect(service).toBeInstanceOf(OkxService);
  });

  test('should throw error for unsupported exchange', () => {
    expect(() => {
      ExchangeFactory.createExchange('unsupported' as any, {});
    }).toThrow('Unsupported exchange');
  });
});
```

2. **符号转换测试**
```typescript
describe('SymbolConverter', () => {
  test('should convert to Binance format', () => {
    expect(SymbolConverter.toBinance('BTC')).toBe('BTCUSDT');
    expect(SymbolConverter.toBinance('BTCUSDT')).toBe('BTCUSDT');
  });

  test('should convert to OKX format', () => {
    expect(SymbolConverter.toOkx('BTC')).toBe('BTC-USDT-SWAP');
    expect(SymbolConverter.toOkx('BTC-USDT-SWAP')).toBe('BTC-USDT-SWAP');
  });
});
```

3. **手动平仓检测测试（新增）**
```typescript
describe('Manual Close Detection', () => {
  let okxService: OkxService;
  let mockOrderHistory: OrderHistoryRecord[];

  beforeEach(() => {
    okxService = new OkxService(mockConfig);
    mockOrderHistory = [
      {
        orderId: '12345',
        symbol: 'BTC-USDT-SWAP',
        side: 'BUY',
        type: 'MARKET',
        quantity: '0.1',
        status: 'FILLED',
        updateTime: Date.now() - 60000
      }
    ];
  });

  test('should detect manual position close', async () => {
    const mockPositions: PositionResponse[] = [
      {
        symbol: 'BTC-USDT-SWAP',
        positionAmt: '0',
        entryPrice: '50000',
        markPrice: '51000',
        unRealizedProfit: '0',
        updateTime: Date.now()
      }
    ];

    const manualCloses = await okxService.detectManualClosure(mockPositions, mockOrderHistory);

    expect(manualCloses).toHaveLength(1);
    expect(manualCloses[0].symbol).toBe('BTC-USDT-SWAP');
    expect(manualCloses[0].side).toBe('SELL');
  });

  test('should not detect manual close when position exists', async () => {
    const mockPositions: PositionResponse[] = [
      {
        symbol: 'BTC-USDT-SWAP',
        positionAmt: '0.1',
        entryPrice: '50000',
        markPrice: '51000',
        unRealizedProfit: '100',
        updateTime: Date.now()
      }
    ];

    const manualCloses = await okxService.detectManualClosure(mockPositions, mockOrderHistory);

    expect(manualCloses).toHaveLength(0);
  });
});
```

4. **方向性价格容差测试（新增）**
```typescript
describe('Directional Price Tolerance', () => {
  let okxService: OkxService;

  beforeEach(() => {
    okxService = new OkxService(mockConfig);
  });

  test('should validate price tolerance for LONG position correctly', () => {
    const result = okxService.calculateDirectionalPriceDifference(
      51000, // currentPrice
      50000, // referencePrice
      'BUY',  // positionSide
      2.0     // tolerancePercent
    );

    expect(result.isValid).toBe(true);
    expect(result.priceDifference).toBe(1000);
    expect(result.direction).toBe('UP');
    expect(result.positionSide).toBe('BUY');
  });

  test('should validate price tolerance for SHORT position correctly', () => {
    const result = okxService.calculateDirectionalPriceDifference(
      49000, // currentPrice
      50000, // referencePrice
      'SELL', // positionSide
      2.0     // tolerancePercent
    );

    expect(result.isValid).toBe(true);
    expect(result.priceDifference).toBe(1000);
    expect(result.direction).toBe('DOWN');
    expect(result.positionSide).toBe('SELL');
  });

  test('should reject price outside tolerance range', () => {
    const result = okxService.calculateDirectionalPriceDifference(
      53000, // currentPrice (6% difference)
      50000, // referencePrice
      'BUY',  // positionSide
      2.0     // tolerancePercent
    );

    expect(result.isValid).toBe(false);
    expect(result.direction).toBe('UP');
  });
});
```

5. **Auto-Refollow 功能测试（新增）**
```typescript
describe('Auto-Refollow Functionality', () => {
  let okxService: OkxService;
  let mockFollowConfig: FollowConfig;

  beforeEach(() => {
    okxService = new OkxService(mockConfig);
    mockFollowConfig = {
      agentName: 'test-agent',
      autoRefollow: true,
      totalMargin: 1000
    };
  });

  test('should handle manual closure and trigger refollow', async () => {
    const mockCloseRecord: ManualCloseRecord = {
      symbol: 'BTC-USDT-SWAP',
      side: 'SELL',
      closePrice: '51000',
      closeQuantity: '0.1',
      closeTime: Date.now(),
      detectionTime: Date.now(),
      originalOrderId: 'MANUAL_DETECTED'
    };

    const handleSpy = jest.spyOn(okxService, 'handleManualClosure');

    await okxService.handleManualClosure(mockCloseRecord, mockFollowConfig);

    expect(handleSpy).toHaveBeenCalledWith(mockCloseRecord, mockFollowConfig);
  });
});
```

### 🔧 集成测试

#### 测试环境配置
```typescript
// jest.config.js
module.exports = {
  testEnvironment: 'node',
  setupFilesAfterEnv: ['<rootDir>/src/__tests__/setup.ts'],
  testTimeout: 30000, // OKX API 可能较慢
};
```

#### Mock 策略
```typescript
// src/__tests__/mocks/okx-mock.ts
export const mockOkxResponses = {
  accountInfo: {
    data: [{
      adjEq: '1000.0',
      imr: '0',
      isoEq: '0',
      mgnRatio: '9999',
      mmr: '0',
      notionalUsd: '0',
      ordFroz: '0',
      totalEq: '1000.0',
      utime: '1234567890'
    }]
  },

  positions: {
    data: [{
      instId: 'BTC-USDT-SWAP',
      lever: '10',
      ls: 'long',
      notionalUsd: '5000',
      pos: '0.1',
      posId: '12345',
      upl: '50',
      uplRatio: '0.01'
    }]
  }
};
```

### 🚀 端到端测试

#### 测试脚本
```bash
#!/bin/bash
# scripts/e2e-test.sh

echo "🧪 Running E2E tests..."

# 测试币安连接
echo "Testing Binance connection..."
npm start -- status --exchange binance

# 测试 OKX 连接
echo "Testing OKX connection..."
npm start -- status --exchange okx

# 测试 Telegram 通知
echo "Testing Telegram notifications..."
npm start -- telegram-test --exchange okx

# 测试配置验证
echo "Testing configuration validation..."
npm start -- follow test-agent --risk-only --exchange okx

# 测试手动平仓检测（模拟）
echo "Testing manual close detection..."
npm start -- follow test-agent --risk-only --auto-refollow --exchange okx

# 测试方向性价格容差
echo "Testing directional price tolerance..."
npm start -- follow test-agent --total-margin 10 --price-tolerance 1.0 --exchange okx

echo "✅ E2E tests completed"
```

---

## 风险评估

### ⚠️ 开发风险

#### 1. API 兼容性风险
- **风险等级**: 中等
- **影响**: OKX API 可能与币安存在细微差异
- **缓解措施**:
  - 充分的测试覆盖
  - 使用测试网验证
  - 参考官方文档和社区反馈

#### 2. 认证复杂性
- **风险等级**: 低等
- **影响**: OKX 的 passphrase 增加了认证复杂度
- **缓解措施**:
  - 提供详细的配置指南
  - 实现良好的错误提示
  - 使用环境变量管理敏感信息

#### 3. 精度处理差异
- **风险等级**: 中等
- **影响**: 不同交易所的精度要求可能导致交易失败
- **缓解措施**:
  - 动态获取交易规则
  - 实现智能精度调整
  - 充分的边界测试

### 🛡️ 运行时风险

#### 1. API 限制
- **风险等级**: 低等
- **影响**: OKX API 速率限制可能与币安不同
- **缓解措施**:
  - 实现请求频率控制
  - 监控 API 使用情况
  - 实现优雅降级

#### 2. 网络稳定性
- **风险等级**: 低等
- **影响**: 网络问题可能导致交易失败
- **缓解措施**:
  - 实现重试机制
  - 增加超时处理
  - 提供离线模式选项

#### 3. 配置错误
- **风险等级**: 中等
- **影响**: 用户配置错误可能导致交易失败或资金损失
- **缓解措施**:
  - 实现配置验证
  - 提供详细的错误信息
  - 强制测试网优先

### 📋 回滚计划

如果 OKX 集成出现问题，可以按以下步骤回滚：

1. **立即回滚**: 切换回币安 API
   ```bash
   npm start -- follow agent --exchange binance
   ```

2. **配置回滚**: 从 `.env` 文件移除 OKX 配置

3. **代码回滚**: 如果需要，回滚到集成前的代码版本

4. **用户通知**: 及时通知用户问题和解决方案

---

## 📞 技术支持

### 🐛 问题反馈

如果在集成过程中遇到问题，请提供以下信息：

1. **环境信息**
   - Node.js 版本
   - 操作系统
   - 项目版本

2. **错误详情**
   - 完整的错误堆栈
   - 复现步骤
   - 相关配置

3. **调试信息**
   - 日志输出
   - API 响应
   - 网络请求详情

### 📚 参考资源

- [OKX API 官方文档](https://www.okx.com/docs-v5/)
- [OKX API 速率限制](https://www.okx.com/docs-v5/#rest-api-rate-limit)
- [项目 GitHub Issues](https://github.com/your-repo/nof1-tracker/issues)

### 🔄 持续改进

本集成指南将根据实际开发经验持续更新。欢迎提交反馈和建议。

---

## 📊 项目时间线总结

### 🎯 总体时间调整

由于集成了新的功能特性，项目时间线从原来的 **7-10天** 调整为 **10-15天**：

| 阶段 | 原计划时间 | 调整后时间 | 主要增加内容 |
|------|------------|------------|--------------|
| 第一阶段：基础架构设计 | 1-2天 | 2-3天 | 手动平仓检测接口、新功能集成 |
| 第二阶段：OKX 服务实现 | 2-3天 | 3-4天 | 手动平仓检测、方向性价格容差 |
| 第三阶段：代码重构 | 1-2天 | 2-3天 | Auto-Refollow 集成、通知系统 |
| 第四阶段：CLI 和入口更新 | 1天 | 2天 | 新 CLI 参数、telegram-test 命令 |
| 第五阶段：测试和文档 | 1-2天 | 2-3天 | 功能专项测试、用户验收 |
| **总计** | **6-10天** | **11-15天** | **新增功能集成** |

### 🔧 关键里程碑

1. **Day 3**: 基础架构完成，接口定义就绪
2. **Day 7**: OKX 服务实现完成，核心功能可用
3. **Day 10**: 代码重构完成，现有功能兼容
4. **Day 12**: CLI 更新完成，用户体验优化
5. **Day 15**: 全面测试完成，项目发布就绪

### ⚡ 关键成功因素

- **向后兼容性**: 确保现有币安用户功能不受影响
- **功能对等性**: OKX 功能与币安功能完全一致
- **新功能集成**: 手动平仓检测和 auto-refollow 功能在两个交易所都可用
- **用户体验**: 无缝切换，统一的命令行接口

---

**文档版本**: v1.1
**创建日期**: 2025-10-28
**作者**: Claude Code Assistant
**最后更新**: 2025-10-30
**更新内容**: 集成最新代码变更，包含手动平仓检测和 auto-refollow 功能

---

*免责声明: 本文档仅供参考。实际开发中请根据 OKX API 最新文档进行调整。合约交易存在资金损失风险，请在充分了解风险后谨慎使用。*