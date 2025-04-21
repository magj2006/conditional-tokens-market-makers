# 多代币池环境中的Tick系统实现方案

## 1. 背景与挑战

当前的条件代币市场使用固定乘积市场做市商(FPMM)模型，该模型维护多个代币池，每个池对应一个可能的结果。与传统的Uniswap V3仅处理两种代币不同，我们面临的是N个代币池的复杂环境。

将Tick系统引入多代币池环境面临以下主要挑战：

1. **价格定义问题**：在多池环境中如何定义和解释"价格"
2. **多池相互关系**：如何处理多个代币池之间的依赖关系
3. **Tick系统适配**：如何将为双代币池设计的Tick系统扩展到多代币池
4. **效率与Gas成本**：如何在复杂环境中保持高效率和可接受的Gas成本

## 2. 基础设计方案

### 2.1 多池环境中的价格定义

在多代币池中，我们需要明确定义"价格"的含义：

```solidity
// 多代币池环境中定义"价格"
// 价格定义为：特定结果代币相对于抵押品的价格
function getPrice(uint outcomeIndex) public view returns (uint) {
    uint[] memory poolBalances = getPoolBalances();
    uint totalBalance = 0;
    
    for (uint i = 0; i < poolBalances.length; i++) {
        totalBalance = totalBalance.add(poolBalances[i]);
    }
    
    // 价格定义为该结果在总池中的占比
    // 使用18位小数表示
    return poolBalances[outcomeIndex].mul(ONE).div(totalBalance);
}
```

### 2.2 价格与Tick的转换

需要建立价格和Tick之间的映射关系：

```solidity
// 价格转Tick
function priceToTick(uint price) public pure returns (int24) {
    // 使用Uniswap V3的方法将价格转换为Tick
    // price需要是18位小数表示
    return TickMath.getTickAtSqrtRatio(
        uint160(sqrt((price << 192) / (ONE - price)))
    );
}

// Tick转价格
function tickToPrice(int24 tick) public pure returns (uint) {
    uint160 sqrtPriceX96 = TickMath.getSqrtRatioAtTick(tick);
    uint256 price = FullMath.mulDiv(
        uint256(sqrtPriceX96).mul(sqrtPriceX96), 
        ONE, 
        1 << 192
    );
    return price;
}
```

### 2.3 数据结构设计

为支持多池环境下的Tick系统，需要设计专门的数据结构：

```solidity
// 为每个结果代币维护单独的Tick订单簿
mapping(uint => mapping(int24 => Order[])) public buyOrdersByOutcomeAndTick;
mapping(uint => mapping(int24 => Order[])) public sellOrdersByOutcomeAndTick;

// 记录每个结果的最新Tick
mapping(uint => int24) public lastTickByOutcome;

// 订单结构
struct Order {
    address owner;           // 订单拥有者
    uint outcomeIndex;       // 结果索引
    uint amount;             // 订单金额（原始抵押品精度）
    uint minReturn;          // 最小返回金额（卖单用）
    int24 tick;              // 限价对应的Tick
    uint expiry;             // 过期时间
    bool isBuy;              // 买单或卖单
    bool active;             // 订单是否活跃
    uint filled;             // 已成交金额
}

// 流动性提供相关结构
mapping(address => mapping(uint => mapping(int24 => mapping(int24 => uint)))) public liquidityByProviderAndTickRange;
mapping(uint => mapping(int24 => mapping(int24 => uint))) public totalLiquidityByTickRange;
```

## 3. 核心功能实现

### 3.1 下限价订单功能

```solidity
// 下买入限价单
function placeBuyLimitOrder(
    uint outcomeIndex, 
    uint amount, 
    uint price,        // 18位小数表示的价格
    uint expiry
) external returns (uint orderId) {
    // 验证参数
    require(outcomeIndex < positionIds.length, "Invalid outcome index");
    require(amount > 0, "Amount must be positive");
    require(price > 0 && price < ONE, "Invalid price"); // 价格在0-1之间
    
    // 转换价格为Tick
    int24 limitTick = priceToTick(price);
    
    // 创建订单ID
    orderId = nextOrderId++;
    
    // 创建订单
    Order memory order = Order({
        owner: msg.sender,
        outcomeIndex: outcomeIndex,
        amount: amount,
        minReturn: 0,         // 买单不需要
        tick: limitTick,
        expiry: expiry > 0 ? block.timestamp + expiry : 0,
        isBuy: true,
        active: true,
        filled: 0
    });
    
    // 存储订单
    orders[orderId] = order;
    buyOrdersByOutcomeAndTick[outcomeIndex][limitTick].push(orderId);
    
    // 转移抵押品
    require(collateralToken.transferFrom(msg.sender, address(this), amount), "Transfer failed");
    
    // 检查是否可以立即执行
    uint currentPrice = getPrice(outcomeIndex);
    if (price >= currentPrice) {
        executeBuyOrder(orderId);
    }
    
    emit OrderPlaced(orderId, msg.sender, outcomeIndex, price, limitTick, amount, true);
    
    return orderId;
}

// 下卖出限价单
function placeSellLimitOrder(
    uint outcomeIndex, 
    uint amount, 
    uint price,        // 18位小数表示的价格
    uint minReturn,    // 最小获得的抵押品
    uint expiry
) external returns (uint orderId) {
    // 验证参数
    require(outcomeIndex < positionIds.length, "Invalid outcome index");
    require(amount > 0, "Amount must be positive");
    require(price > 0 && price < ONE, "Invalid price"); // 价格在0-1之间
    
    // 转换价格为Tick
    int24 limitTick = priceToTick(price);
    
    // 创建订单ID
    orderId = nextOrderId++;
    
    // 创建订单
    Order memory order = Order({
        owner: msg.sender,
        outcomeIndex: outcomeIndex,
        amount: amount,
        minReturn: minReturn,
        tick: limitTick,
        expiry: expiry > 0 ? block.timestamp + expiry : 0,
        isBuy: false,
        active: true,
        filled: 0
    });
    
    // 存储订单
    orders[orderId] = order;
    sellOrdersByOutcomeAndTick[outcomeIndex][limitTick].push(orderId);
    
    // 转移ERC1155代币
    conditionalTokens.safeTransferFrom(
        msg.sender,
        address(this),
        positionIds[outcomeIndex],
        amount,
        ""
    );
    
    // 检查是否可以立即执行
    uint currentPrice = getPrice(outcomeIndex);
    if (price <= currentPrice) {
        executeSellOrder(orderId);
    }
    
    emit OrderPlaced(orderId, msg.sender, outcomeIndex, price, limitTick, amount, false);
    
    return orderId;
}
```

### 3.2 订单执行功能

```solidity
// 执行买入订单
function executeBuyOrder(uint orderId) public {
    Order storage order = orders[orderId];
    require(order.active, "Order not active");
    require(order.isBuy, "Not a buy order");
    require(order.expiry == 0 || block.timestamp <= order.expiry, "Order expired");
    
    // 获取当前价格
    uint currentPrice = getPrice(order.outcomeIndex);
    uint orderPrice = tickToPrice(order.tick);
    
    // 验证价格条件
    require(currentPrice <= orderPrice, "Price condition not met");
    
    // 计算可获得的结果代币数量
    uint outcomeTokensToBuy = calcBuyAmount(order.amount, order.outcomeIndex);
    
    // 设置最小接收数量（添加1%滑点保护）
    uint minOutcomeTokens = outcomeTokensToBuy.mul(99).div(100);
    
    // 执行买入
    // 注意：内部会处理fees和split操作
    try this.buy(order.amount, order.outcomeIndex, minOutcomeTokens) returns (uint boughtAmount) {
        // 将结果代币转给订单拥有者
        conditionalTokens.safeTransferFrom(
            address(this),
            order.owner,
            positionIds[order.outcomeIndex],
            boughtAmount,
            ""
        );
        
        // 更新订单状态
        order.active = false;
        order.filled = order.amount;
        
        // 移除订单从活跃列表
        _removeOrderFromTick(order.outcomeIndex, order.tick, orderId, true);
        
        emit OrderExecuted(orderId, boughtAmount, currentPrice);
    } catch {
        // 处理执行失败的情况
        // 可能是因为市场条件变化、滑点过大等
        revert("Order execution failed");
    }
}

// 执行卖出订单
function executeSellOrder(uint orderId) public {
    Order storage order = orders[orderId];
    require(order.active, "Order not active");
    require(!order.isBuy, "Not a sell order");
    require(order.expiry == 0 || block.timestamp <= order.expiry, "Order expired");
    
    // 获取当前价格
    uint currentPrice = getPrice(order.outcomeIndex);
    uint orderPrice = tickToPrice(order.tick);
    
    // 验证价格条件
    require(currentPrice >= orderPrice, "Price condition not met");
    
    // 计算期望获得的抵押品金额
    uint returnAmount = calcSellReturnAmount(order.amount, order.outcomeIndex);
    
    // 验证最小返回金额
    require(returnAmount >= order.minReturn, "Return amount too low");
    
    // 授权市场合约使用代币
    conditionalTokens.setApprovalForAll(address(this), true);
    
    // 执行卖出
    try this.sell(returnAmount, order.outcomeIndex, order.amount) returns (uint soldAmount) {
        // 确保实际卖出金额不超过订单金额
        require(soldAmount <= order.amount, "Sold amount exceeds order");
        
        // 将抵押品转给订单拥有者
        require(collateralToken.transfer(order.owner, returnAmount), "Transfer failed");
        
        // 更新订单状态
        order.active = false;
        order.filled = soldAmount;
        
        // 移除订单从活跃列表
        _removeOrderFromTick(order.outcomeIndex, order.tick, orderId, false);
        
        emit OrderExecuted(orderId, returnAmount, currentPrice);
    } catch {
        // 处理执行失败的情况
        revert("Order execution failed");
    }
}
```

### 3.3 价格变动监控与触发执行

```solidity
// 在交易后检查订单执行
function _afterTrade(uint outcomeIndex) internal {
    // 获取新价格
    uint newPrice = getPrice(outcomeIndex);
    
    // 转换为Tick
    int24 newTick = priceToTick(newPrice);
    int24 oldTick = lastTickByOutcome[outcomeIndex];
    
    // 如果Tick发生变化，检查可执行的订单
    if (newTick != oldTick) {
        // 价格上升：检查卖单
        if (newTick > oldTick) {
            _checkSellOrders(outcomeIndex, oldTick, newTick);
        } 
        // 价格下降：检查买单
        else {
            _checkBuyOrders(outcomeIndex, newTick, oldTick);
        }
        
        // 更新最新Tick
        lastTickByOutcome[outcomeIndex] = newTick;
    }
}

// 检查可执行的买单
function _checkBuyOrders(uint outcomeIndex, int24 newTick, int24 oldTick) internal {
    // 从高到低遍历Tick（价格下降，买单可能被触发）
    for (int24 tick = oldTick; tick > newTick; tick -= TICK_SPACING) {
        uint[] storage orderIds = buyOrdersByOutcomeAndTick[outcomeIndex][tick];
        
        for (uint i = 0; i < orderIds.length; i++) {
            uint orderId = orderIds[i];
            if (orders[orderId].active) {
                try this.executeBuyOrder(orderId) {
                    // 订单执行成功
                } catch {
                    // 订单执行失败，继续下一个
                }
            }
        }
    }
}

// 检查可执行的卖单
function _checkSellOrders(uint outcomeIndex, int24 oldTick, int24 newTick) internal {
    // 从低到高遍历Tick（价格上升，卖单可能被触发）
    for (int24 tick = oldTick; tick < newTick; tick += TICK_SPACING) {
        uint[] storage orderIds = sellOrdersByOutcomeAndTick[outcomeIndex][tick];
        
        for (uint i = 0; i < orderIds.length; i++) {
            uint orderId = orderIds[i];
            if (orders[orderId].active) {
                try this.executeSellOrder(orderId) {
                    // 订单执行成功
                } catch {
                    // 订单执行失败，继续下一个
                }
            }
        }
    }
}

// 修改现有buy函数以包含订单检查
function buy(uint investmentAmount, uint outcomeIndex, uint minOutcomeTokensToBuy) external returns (uint) {
    // 原有buy逻辑
    uint outcomeTokensToBuy = calcBuyAmount(investmentAmount, outcomeIndex);
    require(outcomeTokensToBuy >= minOutcomeTokensToBuy, "minimum buy amount not reached");

    require(collateralToken.transferFrom(msg.sender, address(this), investmentAmount), "cost transfer failed");

    uint feeAmount = investmentAmount.mul(fee) / ONE;
    feePoolWeight = feePoolWeight.add(feeAmount);
    uint investmentAmountMinusFees = investmentAmount.sub(feeAmount);
    require(collateralToken.approve(address(conditionalTokens), investmentAmountMinusFees), "approval for splits failed");
    splitPositionThroughAllConditions(investmentAmountMinusFees);

    conditionalTokens.safeTransferFrom(address(this), msg.sender, positionIds[outcomeIndex], outcomeTokensToBuy, "");

    emit FPMMBuy(msg.sender, investmentAmount, feeAmount, outcomeIndex, outcomeTokensToBuy);
    
    // 新增：检查订单执行
    _afterTrade(outcomeIndex);
    
    return outcomeTokensToBuy;
}

// 同样修改sell函数以包含订单检查
function sell(uint returnAmount, uint outcomeIndex, uint maxOutcomeTokensToSell) external returns (uint) {
    // 原有sell逻辑
    // ...
    
    // 新增：检查订单执行
    _afterTrade(outcomeIndex);
    
    return outcomeTokensToSell;
}
```

## 4. 高级功能与优化

### 4.1 批量订单处理

为提高效率，实现批量处理订单的功能：

```solidity
// 批量执行买入订单
function batchExecuteBuyOrders(uint outcomeIndex, int24 fromTick, int24 toTick) external {
    require(fromTick >= toTick, "Invalid tick range for buy orders");
    
    uint currentPrice = getPrice(outcomeIndex);
    int24 currentTick = priceToTick(currentPrice);
    
    // 只处理价格条件已满足的区间
    if (toTick <= currentTick) {
        for (int24 tick = fromTick; tick >= toTick; tick -= TICK_SPACING) {
            uint[] storage orderIds = buyOrdersByOutcomeAndTick[outcomeIndex][tick];
            
            for (uint i = 0; i < orderIds.length; i++) {
                uint orderId = orderIds[i];
                if (orders[orderId].active) {
                    try this.executeBuyOrder(orderId) {
                        // 订单执行成功
                    } catch {
                        // 继续执行下一个订单
                    }
                }
            }
        }
    }
}

// 批量执行卖出订单
function batchExecuteSellOrders(uint outcomeIndex, int24 fromTick, int24 toTick) external {
    require(fromTick <= toTick, "Invalid tick range for sell orders");
    
    uint currentPrice = getPrice(outcomeIndex);
    int24 currentTick = priceToTick(currentPrice);
    
    // 只处理价格条件已满足的区间
    if (fromTick >= currentTick) {
        for (int24 tick = fromTick; tick <= toTick; tick += TICK_SPACING) {
            uint[] storage orderIds = sellOrdersByOutcomeAndTick[outcomeIndex][tick];
            
            for (uint i = 0; i < orderIds.length; i++) {
                uint orderId = orderIds[i];
                if (orders[orderId].active) {
                    try this.executeSellOrder(orderId) {
                        // 订单执行成功
                    } catch {
                        // 继续执行下一个订单
                    }
                }
            }
        }
    }
}
```

### 4.2 基于Tick范围的流动性提供

支持基于Tick范围的定向流动性提供：

```solidity
// 在特定Tick范围内添加流动性
function addLiquidityInTickRange(
    uint[] calldata amounts,
    int24[] calldata lowerTicks,
    int24[] calldata upperTicks
) external {
    require(amounts.length == positionIds.length, "Length mismatch");
    require(lowerTicks.length == positionIds.length, "Length mismatch");
    require(upperTicks.length == positionIds.length, "Length mismatch");
    
    // 计算总投入
    uint totalInvestment = 0;
    for (uint i = 0; i < amounts.length; i++) {
        totalInvestment = totalInvestment.add(amounts[i]);
    }
    
    // 转移抵押品
    require(collateralToken.transferFrom(msg.sender, address(this), totalInvestment), "Transfer failed");
    
    // 为每个池添加流动性并记录Tick范围
    for (uint i = 0; i < amounts.length; i++) {
        // 验证Tick范围有效
        require(lowerTicks[i] < upperTicks[i], "Invalid tick range");
        
        // 记录流动性提供者在特定Tick范围内的份额
        liquidityByProviderAndTickRange[msg.sender][i][lowerTicks[i]][upperTicks[i]] = 
            liquidityByProviderAndTickRange[msg.sender][i][lowerTicks[i]][upperTicks[i]].add(amounts[i]);
            
        // 更新总流动性
        totalLiquidityByTickRange[i][lowerTicks[i]][upperTicks[i]] = 
            totalLiquidityByTickRange[i][lowerTicks[i]][upperTicks[i]].add(amounts[i]);
    }
    
    // 执行现有的添加流动性操作
    addFunding(totalInvestment, amounts);
}

// 从特定Tick范围内移除流动性
function removeLiquidityFromTickRange(
    uint outcomeIndex,
    int24 lowerTick,
    int24 upperTick,
    uint amount
) external {
    // 验证用户在该范围内有足够的流动性
    require(
        liquidityByProviderAndTickRange[msg.sender][outcomeIndex][lowerTick][upperTick] >= amount,
        "Insufficient liquidity"
    );
    
    // 更新用户流动性记录
    liquidityByProviderAndTickRange[msg.sender][outcomeIndex][lowerTick][upperTick] = 
        liquidityByProviderAndTickRange[msg.sender][outcomeIndex][lowerTick][upperTick].sub(amount);
        
    // 更新总流动性
    totalLiquidityByTickRange[outcomeIndex][lowerTick][upperTick] = 
        totalLiquidityByTickRange[outcomeIndex][lowerTick][upperTick].sub(amount);
    
    // 计算移除的份额
    uint sharesAmount = amount.mul(totalSupply()).div(getPoolBalances()[outcomeIndex]);
    
    // 执行现有的移除流动性操作
    removeFunding(sharesAmount);
}
```

## 5. 与现有FPMM合约的集成

### 5.1 扩展现有合约

将Tick系统作为现有FPMM合约的扩展实现：

```solidity
// 扩展现有FPMM合约的Hook合约
contract TickBasedLimitOrderHook is ERC1155TokenReceiver {
    // 指向FPMM合约的引用
    FixedProductMarketMaker public marketMaker;
    ConditionalTokens public conditionalTokens;
    
    // Tick系统相关状态变量
    // ...
    
    constructor(address _marketMaker) public {
        marketMaker = FixedProductMarketMaker(_marketMaker);
        conditionalTokens = marketMaker.conditionalTokens();
        
        // 初始化所有结果的初始Tick
        for (uint i = 0; i < marketMaker.positionIds().length; i++) {
            uint price = getPrice(i);
            lastTickByOutcome[i] = priceToTick(price);
        }
    }
    
    // 实现Tick系统相关功能
    // ...
    
    // 从FPMM合约调用的回调
    function onTradeExecuted(uint outcomeIndex) external {
        require(msg.sender == address(marketMaker), "Unauthorized");
        _afterTrade(outcomeIndex);
    }
    
    // 实现ERC1155TokenReceiver接口
    function onERC1155Received(
        address operator,
        address from,
        uint256 id,
        uint256 value,
        bytes calldata data
    ) external override returns (bytes4) {
        // 验证发送者
        require(msg.sender == address(conditionalTokens), "Invalid sender");
        
        // 处理接收到的代币
        // ...
        
        return this.onERC1155Received.selector;
    }
    
    function onERC1155BatchReceived(
        address operator,
        address from,
        uint256[] calldata ids,
        uint256[] calldata values,
        bytes calldata data
    ) external override returns (bytes4) {
        // 验证发送者
        require(msg.sender == address(conditionalTokens), "Invalid sender");
        
        // 处理接收到的代币
        // ...
        
        return this.onERC1155BatchReceived.selector;
    }
}
```

### 5.2 升级方案

为现有部署的FPMM合约添加Tick系统的升级路径：

1. **部署Hook合约**：为每个现有FPMM合约部署对应的TickBasedLimitOrderHook合约
2. **添加事件转发机制**：在FPMM合约中添加事件转发到Hook合约
3. **用户迁移**：鼓励用户使用新的接口下单

## 6. 前端集成及用户体验

### 6.1 前端显示建议

为用户提供清晰的价格与Tick信息：

1. **价格显示**：以百分比形式显示（例如，0.75显示为75%胜率）
2. **Tick显示**：可选显示底层Tick值，便于高级用户参考
3. **历史价格图表**：基于Tick划分的历史价格走势图

### 6.2 用户接口设计

简化用户操作流程：

1. **限价单界面**：允许用户以百分比直接输入限价
2. **滑点控制**：提供简单的滑点控制选项
3. **订单管理**：集中展示所有活跃订单和历史订单
4. **价格预测**：基于当前池状态模拟不同交易量对价格的影响

## 7. 测试与部署策略

### 7.1 测试用例设计

全面测试Tick系统在多池环境中的表现：

1. **基本功能测试**：验证限价单下单、取消和执行的基本功能
2. **价格边界测试**：测试极端价格情况下的系统行为
3. **交互测试**：验证多个用户同时操作的行为
4. **Gas成本测试**：测量不同操作的Gas成本
5. **压力测试**：验证系统在高负载下的表现

### 7.2 分阶段部署策略

1. **Phase 1 - 基础功能**：部署基本的Tick系统，支持简单的限价单操作
2. **Phase 2 - 高级功能**：添加批量处理和基于Tick范围的流动性功能
3. **Phase 3 - 优化与扩展**：根据用户反馈进行优化，添加更多高级功能

## 8. 总结与展望

### 8.1 主要优势

1. **精确价格控制**：通过Tick系统提供精确的价格控制能力
2. **效率提升**：批量处理和订单自动执行提高了效率
3. **流动性优化**：基于Tick范围的流动性提供提高了资金效率
4. **用户体验增强**：限价单功能极大改善了交易体验

### 8.2 潜在挑战

1. **复杂性**：多池环境中的Tick系统比双代币池更复杂
2. **Gas成本**：需要持续优化以控制Gas成本
3. **用户理解**：需要帮助用户理解多池环境中的价格概念

### 8.3 未来发展方向

1. **多级Tick间隔**：支持不同粒度的Tick间隔
2. **自动做市策略**：基于Tick系统的高级做市策略
3. **跨池策略**：支持跨多个结果池的复杂交易策略
4. **交互式图表**：提供更直观的价格与流动性可视化

通过将Tick系统引入多代币池环境，条件代币市场将获得显著的功能提升，为用户提供更精确、更灵活的交易体验，同时保持固定乘积模型的基本优势。 