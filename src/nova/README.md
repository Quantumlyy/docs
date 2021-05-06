# Nova

**Nova is a <u>set of contracts</u> & <u>network of bots</u> that empowers users/contracts to seamlessly interact with L1 contracts + liquidity <u>without leaving L2</u> in a trustless and <u>composable</u> manner.**

<img width="500" style="float: right;" alt="Explainer" src="https://i.imgur.com/TbbAhLd.png">

- Users specify what actions they want run on L1 from L2

- Users pay a bounty which pays for the gas of execution on L1 + whatever upfront costs a bot executing on L1 needs to have.

- Bots execute requests on L1 by calling the Nova "Execution Manager" contract with the calldata users on L2 give them.

- The execution manager will call a specific strategy contract which can send tokens up to L2 via a bridge.

- After executing a request, the Nova Execution Manager sends a confirmation up to L2 to unlock the bounty for the bot.

## L2_NovaRegistry

This is the "user facing" contract of Nova. Users can use this contract to [request execution of different strategies](#request-execution), [cancel their requests](#cancel-execution-request), [withdraw their tokens](#withdraw-tip-input-tokens), and [bump the gas price of their requests](#bump-request-gas-price).

Bots also will use this contract to [view the latest requests](#get-all-request-information) and [receive tips for executing requests](#complete-execution-request).

### Request execution

```solidity
struct InputToken {
    address l1Token;
    address l2Token;
    uint256 amount;
}

/// @param strategy The address of the "strategy" contract on L1 a bot should call with `l1calldata`.
/// @param l1calldata The abi encoded calldata a bot should call the `strategy` with on L1.
/// @param gasLimit The gas limit a bot should use on L1.
/// @param gasPrice The gas price a bot should use on L1.
/// @param tip The additional wei to pay as a tip for any bot that executes this request.
/// @param inputTokens An array of token amounts that a bot will need on L1 to execute the request (`l1Token`s) along with the equivalent tokens that will be returned on L2 (`l2Token`s). `inputTokens` will not be awarded if the `strategy` reverts on L1.
function requestExec(address strategy, bytes calldata l1calldata, uint256 gasLimit, uint256 gasPrice, uint256 tip, InputToken[] calldata inputTokens) public returns (bytes32 execHash)
```

This function allows a user to request a strategy to be executed.

It will first increment the contract's nonce (which is to prevent duplicate execution requests from having the same `execHash`) then transfer in all the `InputToken`s (**all must be approved to the registry by the caller**), and the amount of ETH neccessary to pay for the max amount of gas used + the tip.

::: warning
The inputs are not checked to be sufficient by the registry, it is up to Nova bots to determine which requests are profitable via `getRequestData`.
:::

It will then compute the `execHash` (unique identifier of this specific execution request) like so: `keccak256(abi.encodePacked(nonce, strategy, l1calldata, gasPrice))` and store it in in a mapping and assigned to all of the arguments this function was passed.

### Request execution with a timeout

```solidity
function requestExecWithTimeout(address strategy, bytes calldata l1calldata, uint256 gasLimit, uint256 gasPrice, uint256 tip, InputToken[] calldata inputTokens, uint256 autoCancelDelay) external returns (bytes32 execHash)
```

Behaves exactly like `requestExec` but also calls `cancel` with `autoCancelDelay` automatically.

::: warning
The user will still have to call `withdraw` once the `autoCancelDelay` timeout completes.
:::

This function is useful for strategies that are likely to cause hard reverts or not be executed for some reason.

### Cancel execution request

```solidity
function cancel(bytes32 execHash, uint256 withdrawDelaySeconds) public
```

This function cancels an execution request (allows a user to withdraw their tip/inputs after a delay). `msg.sender` must be the initiator of execution request the `execHash` links to.

::: tip
After `cancel` is called the user must wait `withdrawDelaySeconds` before calling `withdraw` to get their tip, input tokens, etc back.
:::

::: warning
`withdrawDelaySeconds` must be >=300 (5 minutes).
:::

A bot can still execute the request associated with the `execHash` up until the withdraw delay has passed.

A user may call may not call `cancel` a second time on the same `execHash`.

### Withdraw tip/input tokens

```solidity
function withdraw(bytes32 execHash) external
```

This function gives the request's creator their input tokens, tip, and gas payment back.

A user cannot call this function unless they have already called `cancel` and waited for at least the `withdrawDelaySeconds` they specified when calling `cancel`.

### Bump request gas price

```solidity
function bumpGas(bytes32 execHash, uint256 gasPrice) external returns (bytes32 newExecHash)
```

`bumpGas` allows a user to increase the gas price for their execution request without having to `cancel`, `withdraw` and call `requestExec` again. Calling this function will initiate a 5 minute delay before disabling the request associated with `execHash` (this is known as the "uncled" request) and enabling an updated version of the request (this is known as the resubmitted request which can be found under `newExecHash`).

::: danger
A bot can still execute the uncled request associated with the `execHash` up until the delay has passed.
:::

If a bot executes the uncled request before the delay has passed the resubmitted request will not be executable after the delay.

### Check if request is executable

```solidity
function isExecutable(bytes32 execHash) public view returns (bool executable, uint256 changeTimestamp)
```

Returns if the request is executable (`executeable`) along with a timestamp of when that may change (`changeTimestamp`).

::: tip
Bots should call this function before trying to execute a request in the registry.
:::

The `changeTimestamp` will be timestamp indicating when the request might switch from being executable to unexecutable (or vice-versa):

- It will be 0 if there is no change expected.
- It will be a timestamp if the request will be enabled soon (as it's a resubmitted version of an uncled request) or the request is being canceled soon.

### Get all request information

```solidity
function getRequestData(bytes32 execHash)
    external
    view
    returns (
        // General request data:
        address strategy,
        bytes memory l1calldata,
        uint256 gasLimit,
        uint256 gasPrice,
        uint256 tip,
        InputToken[] memory inputTokens,
        // Other data:
        uint72 nonce,
        address creator,
        bytes32 uncle,
        // Can be fetched via `isExecutable`:
        bool executable,
        uint256 changeTimestamp
    )
```

Returns all relevant data about a request by its `execHash`.

- The first 6 return items are the parameters passed to `requestExec`.
- `nonce` is the nonce assigned to this request. It is used to compute the `execHash`.
- `creator` is the address which called `requestExec` to create this request.
- `uncle` may either be an empty bytestring or the execHash of the uncle of this transaction (the transaction that this resubmitted transaction is cloned from).
- The last two return items are the return value s of calling `isExecutable` with `execHash`.

### Complete execution request

```solidity
function execCompleted(bytes32 execHash, address rewardRecipient, uint256 gasUsed, bool reverted) external onlyXDomainMessageFromNovaExecutionManager
```

::: danger NOT DIRECTLY CALLABLE
This function can only be called via a message relayed from cross domain messenger with the L1 origin being the `L1_NovaExecutionManager` contract.
:::

The `execHash` gets computed by the `L1_NovaExecutionManager` like so: `keccak256(abi.encodePacked(nonce, strategy, l1calldata, gasPrice))` and is used to ensure the right calldata **(and gas price)** was used on L1.

Once the registry verifies that the `execHash` was previously registered (meaning this execution was valid) & not disabled (via `isDisabled`):

- It will find this `execHash` in the registry's storage and retrieve the `gasPrice` and tip/inputToken information associated with this execHash.

- It will first pay for the gas cost of L1 execution by calculating the ETH to send to the `bot` using `(gasLimit > gasUsed ? gasUsed : gasLimit) * gasPrice`. Any remaining ETH will be sent back to the user who requested execution (just like how gas is refunded on L1 if the gas limit exceeds gas used).

- It will then loop over all the `inputTokens` and transfer the `amount` of each `l2Token` to either:

  1. The `rewardRecipient` if `reverted` is false.
  2. The request's creator if `reverted` is true.

- It will then loop over all the `bounties` and transfer the `amount` of each `l2Token` to the `rewardRecipient`. **If `reverted` is true it will transfer 30% of the amount back to the request's creator and only 70% to the `rewardRecipient`.**

After all the bounties/inputs have been paid out it will mark `execHash` as executed so it cannot be executed again.

---

## L1_NovaExecutionManager

Users on L2 never need to interact with this contract. This contract is to facilitate the execution of requests and send messages to unlock input tokens/tip for bots/executors (post-execution).

Strategy contracts may wish to call back into this contract to trigger a [hard revert](#trigger-hard-revert), [get the current execHash](#get-the-current-exechash) or [transfer tokens from the executor/bot](#transfer-tokens-from-the-executor).

### Execute Request

```solidity
function exec(uint72 nonce, address strategy, bytes memory l1calldata) public
```

This function calls the `strategy` address with the specified `l1calldata`.

The call to `strategy` is wrapped in a try-catch block:

- If the call reverts and the revert message is `__NOVA__HARD__REVERT__`, **`exec` will revert immediately (no message to L2 will be sent).**
  - [This is called a HARD REVERT.](#execute-request)
  - Strategy contracts should only **hard revert** if the bot has not properly set up the execution context (like not approving the right amount input of tokens, etc)
- If the call reverts and the revert message is empty or is not `__NOVA__HARD__REVERT__`, **`exec` will continue with sending a message to L2.**
  - [This is called a SOFT REVERT.](#execute-request)
  - If a strategy **soft reverts**, the `inputTokens` for the request will **not be sent** to the bot and **only 70% of the tip** will be sent (instead of the usual 100%). The **30% tip penalty** is to prevent bots from attempting to cause or wait for soft reverts and **act in good faith** instead.

The `nonce` argument is used to compute the `execHash` needed to unlock the inputs/tip for this strategy on L2.

::: tip
Bots cannot call `exec` with arguments that produce an `execHash` which has previously been successfuly executed.
:::

All computation in the function leading up to the cross domain message is sandwiched between calls to `gasLeft()`. These are used to calculate how many gas units the bot had to pay for (so the registry can **release the proper payment** on L2). However, we are not able to account for refunds so users may end up over-paying their executors (depending on the strategy).

After the call to `strategy` is completed, the EM will compute the `execHash` it needs (using the arguments passed into `exec` along with the `tx.gasprice`) and **send a cross domain message** to call the `L2_NovaRegistry`'s `execCompleted` with the neccessary arguments. This will send the `inputTokens`/`tip` to the caller of `exec` on L2.

```solidity
function execWithRecipient(uint72 nonce, address strategy, bytes calldata l1calldata, address l2Recipient) external
```

Behaves like `exec` but tells the `L2_NovaRegistry` contract to send the `inputTokens`/`tip` to the `l2Recipient` on L2 (instead of specifically the bot who calls the function).

### Trigger Hard Revert

```solidity
function hardRevert() external
```

Convenience function that simply runs `revert("__NOVA__HARD__REVERT__")`.

### Get The Current ExecHash

```solidity
function currentExecHash() external view returns (bytes32)
```

This function returns the execHash computed from the current call to `exec`. Strategy contracts may wish to call this function to send messages up to L2 with and tag them with the current execHash.

### Get The Current Executor

```solidity
function currentExecutor() external view returns (address)
```

This function returns the current "executor" (address that made the current call to `exec`). Strategy contrats may wish to call this function to ensure only a trusted party is able to execute the strategy or to release additional rewards for the executor, etc.

### Transfer Tokens From The Executor

```solidity
function transferFromExecutor(address token, uint256 amount) external
```

This function transfers tokens the calling bot (the account that called `execute`) has approved to the execution manager to the currently executing `strategy`.

::: danger
Only the currently executing `strategy` can call this function.
:::

This function will trigger a [HARD REVERT](#execute-request) if the bot executing the current strategy has not approved at least `amount` of `token` to the `L1_NovaExecutionManager` (like `safeTransferFrom`).

## Example Integration(s)

### Uniswap/Sushiswap

To integrate **Uniswap/Sushiswap** we only need to write one custom contract (a Strategy contract on L1).

- This strategy would have all the same methods as the Uniswap router has
- The `to` parameter of the strategy's methods would be hijacked and not passed into the Uniswap router.
  - The `to` param will be used as the recipient of the tokens on L2.
  - The Uniswap router will be told to send the output tokens back to the `Nova_UniswapStrategy` contract (so it can send them up to L2 via the bridge)
- Each of the methods would require that a bot approve the tokens necessary for the swap to the `L1_NovaExecutionManager`
- The method would call `transferFromBot` to get the input tokens from the bot and then perform the corresponding method call on the Uniswap router.
- The method would then send the output tokens through an Optimism token bridge to the `to` address.

**Here's what one of those wrapped router functions in the Strategy contract would look like:**

```solidity
function swapExactTokensForTokens(
  uint256 amountIn,
  uint256 amountOutMin,
  address[] calldata path,
  address to,
  uint256 deadline
) external {
  ERC20 input = ERC20(path[0]);
  ERC20 output = ERC20(path[path.length - 1]);

  // Transfer in tokens from the bot.
  L1_NovaExecutionManager(msg.sender).transferFromBot(input, amountIn);

  // Approve the input tokens to the uniswapRouter
  input.approve(address(uniswapRouter), amountIn);

  // Perform the swap
  uniswapRouter.swapExactTokensForTokens(
    amountIn,
    amountOutMin,
    path,
    address(this),
    deadline
  );
  uint256 outputAmount = output.balanceOf(address(this));

  // Approve the output tokens to the token bridge
  output.approve(address(optimismTokenBridge), outputAmount);
  // Send the tokens up to L2 with the recipient being the `to` param
  optimismTokenBridge.depositAsERC20(address(output), to, outputAmount);
}

```

## Future MEV Extraction Mechanism

::: danger NOT IMPLEMENTED (YET)
The mechanism explained below is not currently implemented, but will be in a future iteration of Nova.
:::

### Background

An important property of Nova is that it is censorship resistant. There is no single "operator" who can execute requests, anyone is free to. Having a competitive landscape of different bots filling orders is important to ensure users can always get their execution requests filled and they are never censored.

However, considering that many of these requests will come with a tip that makes the request profitable beyond the maximum gas it takes to execute them, it is natural for multiple bots to engage in PGAs to extract profit from as many strategies that they can.

The profits from these PGAs don't go to the Nova platform or users who request execution, they go to **miners** who contribute no value to the protocol.

### MEV Auctions (MEVA)

We can extract the value that would have gone to miners by auctioning off "priority rights" to execute requests for specific strategies (this is also known as a [MEVA](https://ethresear.ch/t/mev-auction-auctioning-transaction-ordering-rights-as-a-solution-to-miner-extractable-value/6788)). Each strategy will have its own sequencer (bot with priority rights) to prevent a sequencer from potentially ignoring a strategy that their bot is not capable of fulfilling executions for. The auctions will function like so:

- Every X hours (configurable) anyone would be able to call `function triggerAuction(address strategy)` on the `L2_NovaRegistry`.
- From there a 5-minute auction would be initiated
- Every bid must be at least 20% greater than the previous bid
- If there is a new bid within the last 1 minute of the auction, the auction timer is extended by 1 minute
- During this 1 minute period the next bid must be at least 40% greater than the previous bid
- If there is another bid in this 1 minute then another 1 minute is added to the timer with the same 40% bid difference requirement **(this repeats until there are no bids in a 1 minute period)**
- The auction winner's bid is taken by the system while all other bids are sent back to their respective bidders.
- The winner is given ownership of an NFT (known as the "priority key") that they can transfer around at will.

::: tip
The owner of the priority key for each strategy will from here on be referred to as a strategy's "sequencer".
:::

The strategy's sequencer is given a Y (configurable) minute window where **only they** can execute that specific strategy. Any other bot performing an execution for a strategy during its "sequencer window" will not receive the execution request's tips (the strategy's sequencer will).

::: tip
After the Y minute window expires for the request any bot is free to execute requests and receive the full tip.
:::

Users will be able to opt out of giving the strategy sequencer priority when requesting an execution (but will pay a small penalty).

### Sequencer Extractable Value

This system not only extracts PGA profits that would have gone to miners, but they are also able to **extract other frontrunning profits** that would have gone to sandwich bots, etc.

- The sequencer effectively has the rights to **reorder transactions** within that 1 minute window
- Importantly, **they can insert their own transactions** inbetween/around them as part of an atomic bundle (via something like a DSProxy).
- Atomic insertion and reoreding rights allow them to take advantage of frontrunning schemes like sandwich attacks without miner/other bot competition.
- Bots bidding in auctions for different strategies will price-in the frontrunning profits they estimate they can extract and adjust their bid accordingly.
  - **Since the profits from these auctions go to the protocol, we have effectively extracted MEV profits that miners/frontrunning bots could have made off of Nova users and brought it back to the protocol instead.**
  - _We can even redistribute the profits we earn from MEVA back to users as a way to reduce costs!_
