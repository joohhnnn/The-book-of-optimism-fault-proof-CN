# op-program

op-program 主要分为两大模块：

1. 提供 ELF 文件的 Client 模块。
2. 启动并维护 pre-image 的 Host 模块。

## Client 模块

原生的基于 op-node 和 op-geth 的实现模块因其复杂性及某些操作无法被 MIPS 表示，无法直接生成对应的 ELF 文件。这就需要一个能完全表达 L2 state 执行、去除复杂操作且完全适配 MIPS 的简化版 op-node 和 op-geth 实现。Client 模块是一个实现了与生产代码逻辑一致的 Golang 模块，用于生成适配 MIPS 使用的 ELF 文件。此 ELF 文件将在后续被 Cannon 解析，生成包含内存、寄存器排列的 state.json 供 Cannon 使用。

### ELF 文件
ELF（Executable and Linkable Format）文件是一种广泛使用的文件格式，主要用于 Unix 和类 Unix 操作系统（如 Linux）。这种格式支持可执行文件、可重定位文件以及共享库，并允许包含详细的元数据，如程序的内存布局、各种段（如代码段、数据段）的位置和大小，以及其他用于动态链接和程序执行的信息。

当程序被编译并打包成一个 ELF 文件时，其内存布局（如代码段和数据段的位置）及启动时需要的初始化设置被固定下来。在 Cannon 中，这些数据被转化为 state.json，作为输入供 Cannon 使用。

### 核心逻辑

[runDerivation()](https://github.com/ethereum-optimism/optimism/blob/develop/op-program/client/program.go#L63) 函数实现了 op-stack 中 L2 state 变化的执行过程。

```
// runDerivation executes the L2 state transition, given a minimal interface to retrieve data.
func runDerivation(logger log.Logger, cfg *rollup.Config, l2Cfg *params.ChainConfig, l1Head common.Hash, l2OutputRoot common.Hash, l2Claim common.Hash, l2ClaimBlockNum uint64, l1Oracle l1.Oracle, l2Oracle l2.Oracle) error {
	l1Source := l1.NewOracleL1Client(logger, l1Oracle, l1Head)
	engineBackend, err := l2.NewOracleBackedL2Chain(logger, l2Oracle, l2Cfg, l2OutputRoot)
	if err != nil {
		return fmt.Errorf("failed to create oracle-backed L2 chain: %w", err)
	}
	l2Source := l2.NewOracleEngine(cfg, logger, engineBackend)

	logger.Info("Starting derivation")
	d := cldr.NewDriver(logger, cfg, l1Source, l2Source, l2ClaimBlockNum)
	for {
		if err = d.Step(context.Background()); errors.Is(err, io.EOF) {
			break
		} else if err != nil {
			return err
		}
	}
	return d.ValidateClaim(l2ClaimBlockNum, eth.Bytes32(l2Claim))
}
```

注意，此模块只负责编译生成 ELF 文件，而将 ELF 文件解析为 state.json 的逻辑是在 Cannon 中完成的。

## Host 模块
当 Cannon 启动时，会同时启动 Host 模块（`./bin/op-program`，这里指的是 host 编译后的二进制名称，不是整个 op-program 模块）。当执行 syscall 时，如果需要读取操作，会抛出一个 hint，Host 的服务捕获此 hint 并将数据加载，以供使用。

因为 Host 是随着 Cannon 的启动命令启动的，因此 Host 是 Cannon 的子模块，二者通过一对 `FileChannel` 对象进行通讯，例如 `pClientRW, pOracleRW`，`pOracleRW` 通道在创建子命令时传递到子程序，并通过如 `pWriter := os.NewFile(6, "pOracleWriter")` 的方式获取。

```
func NewProcessPreimageOracle(name string, args []string) (*ProcessPreimageOracle, error) {
	if name == "" {
		return &ProcessPreimageOracle{}, nil
	}

	pClientRW, pOracleRW, err := preimage.CreateBidirectionalChannel()
	if err != nil {
		return nil, err
	}
	hClientRW, hOracleRW, err := preimage.CreateBidirectionalChannel()
	if err != nil {
		return nil, err
	}

	cmd := exec.Command(name, args...) // nosemgrep
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	cmd.ExtraFiles = []*os.File{
		hOracleRW.Reader(),
		hOracleRW.Writer(),
		pOracleRW.Reader(),
		pOracleRW.Writer(),
	}
```

在子进程代码中使用这些描述符：

```
// CreatePreimageChannel returns a FileChannel for the preimage oracle in a detached context
func CreatePreimageChannel() oppio.FileChannel {
	r := os.NewFile(PClientRFd, "preimage-oracle-read")
	w := os.NewFile(PClientWFd, "preimage-oracle-write")
	return oppio.NewReadWritePair(r, w)
}
```

当通讯建立后，当 Cannon 中需要读取对应 key 的 pre-image 数据后，会往 channel 中写入 key，Host 从 channel 中读取 key，并根据 key 来获取相应的数据，并将数据写入 channel，供 Cannon 使用。

核心函数为 [GetPreimage()](https://github.com/ethereum-optimism/optimism/blob/develop/op-program/host/prefetcher/prefetcher.go#L80) 函数，此函数收到由 Cannon 传过来的 key，并对内容进行获取。

```
func (p *Prefetcher) GetPreimage(ctx context.Context, key common.Hash) ([]byte, error) {
	p.logger.Trace("Pre-image requested", "key", key)
	pre, err := p.kvStore.Get(key)
	// Use a loop to keep retrying the prefetch as long as the key is not found
	// This handles the case where the prefetch downloads a preimage, but it is then deleted unexpectedly
	// before we get to read it.
	for errors.Is(err, kvstore.ErrNotFound) && p.lastHint != "" {
		hint := p.lastHint
		if err := p.prefetch(ctx, hint); err != nil {
			return nil, fmt.Errorf("prefetch failed: %w", err)
		}
		pre, err = p.kvStore.Get(key)
		if err != nil {
			p.logger.Error("Fetched pre-images for last hint but did not find required key", "hint", hint, "key", key)
		}
	}
	return pre, err
}
```

## 总结
- 官方对 [OP-program 的介绍](https://docs.optimism.io/stack/protocol/fault-proofs/fp-components#fault-proof-program) 可以完美总结 Client 模块的功能：该程序通过 rollup 状态转换来验证从 L1 输入得到的 L2 输出。这种可验证的输出随后可以用来解决 L1 上的争议输出。FPP 是 op-node 和 op-geth 的结合体，因此它在单一进程中同时具备了协议的共识和执行“部分”。这意味着通常通过 HTTP 进行的 Engine API 调用改为直接对 op-geth 代码进行方法调用。FPP 设计之初就是为了能够以确定性方式运行，这样使用相同的输入数据进行的两次调用不仅会产生相同的输出，还会产生相同的程序执行迹。这使得它可以作为争议解决过程的一部分，在链上 VM 中运行。

- 对于 Host 模块，它是一个为 Cannon 在运行时提供 Pre-image 信息的服务。
