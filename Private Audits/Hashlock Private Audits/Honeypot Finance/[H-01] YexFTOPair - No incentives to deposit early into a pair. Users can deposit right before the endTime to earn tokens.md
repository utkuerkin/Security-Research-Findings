## Description
There are no incentives in the protocol for early depositors. Users can wait till the
`endTime` to assess the pair and deposit. These users will earn as much rewards as early
depositors. This will lead to no user depositing early which can break protocol
functionality.

## Vulnerability Details
The `deposit` function in the `YexFTOFacede`  contract allows users to deposit `raisedToken`
into the relevant pair.
```solidity
function deposit(
  address raisedToken,
  address launchedToken,
  uint256 raisedTokenAmount
) external override {
  _deposit(
    raisedToken,
    launchedToken,
    raisedTokenAmount
  );
}
```
This allows users to earn LP tokens and `launchedToken`  after the fundraising period
ends, depending on the amount of tokens they have deposited. How much a user has
earned is calculated with the `_calculateLPAmount` and `_calculateLaunchedTokenAmount`
functions shown below:
```solidity
function _calculateLPAmount(address caller) internal view returns (uint256 lpAmount) {
    //comments
    uint256 cumulativeLP = IERC20(lpToken).balanceOf(address(this)) + totalClaimedLp;
    //comments
    lpAmount = cumulativeLP >> 1;
    //comments
    if (launchedTokenProvider != caller) lpAmount = (raisedTokenDeposit[caller] * lpAmount) / depositedRaisedToken;
}

function _calculateLaunchedTokenAmount(address caller) internal view returns (uint256 amount) {
    //comments
    amount = (raisedTokenDeposit[caller] * poolLaunchedTokenAmount) / depositedRaisedToken;
}
```
As observed in these functions, the amount of tokens a user can get is only dependent
on their `raisedTokenDeposit` amount compared to the whole pool. This allows users to
deposit tokens right before the `endTime` and be entitled to as many tokens as the early
depositors.

## Proof of Concept
Implement and run the following test in `hookFTO.ts`. Observe the logs to notice the
vulnerability.
```typescript
it.only("test CustomHook launched FTO burn deposit late", async function () {
    const name = "TestToken";
    const symbol = "TT";
    const amount = ethers.utils.parseUnits("1", 18);
    const poolHandler = henloDexRouter.address;
    const raisingCycle = 120; // 120 seconds
    const launchedPercent = 95;
    const hookPercent = 50; // lock 50% lp
    let startTimestamp =
        (await ethers.provider.getBlock(ethers.provider.blockNumber)).timestamp +
        raisingCycle; // after raised
    const durationSeconds = 10000; // vesting duaration
    const hook_params = ethers.utils.defaultAbiCoder.encode(
        ["uint64", "uint64", "address"],
        [startTimestamp, durationSeconds, owner.address]
    );
    const data = ethers.utils.defaultAbiCoder.encode(
        ["uint256", "bytes"],
        [hookPercent, hook_params]
    );
    // 1. customHook createFTO
    await customHook.createFTO(
        usdt.address,
        name,
        symbol,
        amount,
        launchedPercent,
        poolHandler,
        raisingCycle,
        data
    );
    const ftoPair_addr = await ftoFactory.allPairs(0);
    const YexFTOPair = await ethers.getContractFactory("YexFTOPair");
    const ftoPair = YexFTOPair.attach(ftoPair_addr) as YexFTOPair;
    const launchedToken = await ftoPair.launchedToken();
    const raisedTokenGetter = await customHook.raisedTokenReceiver(ftoPair_addr);
    const launchedTokenProvider = await ftoPair.launchedTokenProvider();
    expect(await ftoPair.percent4hook()).to.be.equal(hookPercent);
    expect(await customHook.beneficiary(ftoPair_addr)).to.be.equal(
        ftoPair_addr
    );
    expect(await customHook.start(ftoPair_addr)).to.be.equal(startTimestamp);
    expect(await customHook.duration(ftoPair_addr)).to.be.equal(
        durationSeconds
    );
    // 2. deposit and perform addr2
    const deposit_amount = ethers.utils.parseUnits("10", 18);
    await usdt.connect(addr2).faucet();
    await usdt.connect(addr2).approve(ftoFacade.address, deposit_amount);
    await ftoFacade
        .connect(addr2)
        .deposit(usdt.address, launchedToken, deposit_amount);
    // increase time to end of fundRaising period
    await network.provider.send("evm_increaseTime", [raisingCycle - 10]);
    await network.provider.send("evm_mine");
    // addr1 deposit
    const deposit_amount2 = ethers.utils.parseUnits("100", 18);
    await usdt.connect(addr1).faucet();
    await usdt.connect(addr1).approve(ftoFacade.address, deposit_amount2);
    await ftoFacade
        .connect(addr1)
        .deposit(usdt.address, launchedToken, deposit_amount2);
    //increase time to end the fundRaising period and perform upkeep
    await network.provider.send("evm_increaseTime", [11]);
    await network.provider.send("evm_mine");
    expect((await ftoPair.checkUpkeep("0x"))[0]).to.be.true;
    await ftoPair.performUpkeep("0x");
    // 3. Logs
    await customHook.connect(addr1).withdrawRaisedToken(ftoPair_addr);
    console.log("Claimable LP addr1:", await
        ftoFacade.connect(addr1).claimableLP(usdt.address, launchedToken));
    console.log("Claimable LP addr2:", await
        ftoFacade.connect(addr2).claimableLP(usdt.address, launchedToken));
    console.log("Claimable Launched of addr1:", await
        ftoFacade.connect(addr1).claimableLaunchedToken(usdt.address, launchedToken));
    console.log("Claimable Launched of addr2:", await
        ftoFacade.connect(addr2).claimableLaunchedToken(usdt.address, launchedToken));
});
```
This test will print the following logs:
```typescript
Claimable LP addr1: BigNumber { value: "2207135896050889450" }
Claimable LP addr2: BigNumber { value: "220713589605088945" }
Claimable Launched of addr1: BigNumber { value: "45454545454545454" }
Claimable Launched of addr2: BigNumber { value: "4545454545454545" }
```

## Impact
There are no incentives to deposit early into a pair’s active time. Users can deposit at
the end and benefit from rewards. This will create a situation in which most users will
wait until the `endTime` to assess the pair. Having no users deposit early can break the
protocol's functionality.

## Recommendation
Implement a time-weighted system when calculating the amount of tokens a user has
earned. Depositing early should allow users to receive more tokens than late depositors.

An example solution could be to split the fundraising period into stages and implement
different multipliers for these stages. For example, a 30-day fundraising period can be
split into 10 stages each lasting 3 days.

Days 0-3 have a multiplier of 1
Days 3-6 have a multiplier of 0.9
…
Days 27-30 have a multiplier of 0.1

Create a new variable to keep track of users' multipliers depending on the stage they
have deposited in and use this multiplier to decide on the amount of tokens they should
earn.
