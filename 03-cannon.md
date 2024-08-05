# CANNON
CANNON 是整个 fault-proof 架构中的核心组件。本章节将对 CANNON 进行详细介绍。

## 组件间关系
在详细介绍 CANNON 之前，我们需要了解一个重要概念：CANNON 并不是独立运行的。CANNON 与 OP-Challenger、OP-Program、OP-Preimage 之间存在协调调用的整体关系。

OP-Challenger 负责总控制，监控链上数据并执行各种操作，如创建游戏、执行 move、step 和 resolve 等。

CANNON 包含两个部分：一部分是链上的 MIPS.sol，这是由 Solidity 编写的 MIPS 指令处理程序，主要负责链上验证最细粒度的执行。另一部分是链下的 mipsevm，由 Go 语言实现，主要用于生成链上所需的 witness 数据，这些数据可以理解为 step 函数的输入参数。链上和链下部分是等效的，即给定相同的输入，二者将产生相同的输出。

OP-Program 也包含两部分：一部分是 client，负责将其自身转化为 ELF 文件，并加载到 CANNON 中使用。另一部分是 host，负责配置并启动 OP-Preimage 服务，供 mipsevm 使用时提供必要的额外数据。

OP-Preimage 负责处理实际的 preimage 逻辑。

他们之间的关系如图所示：
![image](./resources/control.png)

### OP-Challenger 与 Cannon 的交互
当 OP-Challenger 需要生成证明去链上执行 step 操作时，它会运行 CANNON 以获取所需的 state、proof 或 preimage 的额外信息。

### Cannon 与 OP-Program 的交互
1. OP-Program 的 client 部分为 CANNON 提供 ELF 指令集文件，用于 CANNON 的运行。
2. OP-Program 的 host 部分为 CANNON 提供额外数据，如区块号等非原生指令数据。

### OP-Program 与 OP-Preimage 的交互
OP-Program 的 host 部分初始化并配置启动 OP-Preimage，为 CANNON 提供所需的额外数据。


## MIPS
在这里，我们不需要深入了解 MIPS 本身，而是先考虑为什么需要这种中间介质。在我们之前的设计 FDG 中，涉及到在链上执行最细粒度指令以进行验证。简单来说，就是在 L1 的 EVM 环境中使用 Solidity 实现一个 L2 的后端执行客户端。将整个系统完全还原到 Solidity 中是不可能的，即使尽最大努力在 Solidity 中还原，由于复杂性，也难以保证在相同输入的情况下链上链下输出相同的结果。因此，我们需要一种中间态 VM 来确保链下和链上运行的结果是一致的，MIPS 就是这样的存在。在链下，它不直接运行 Go 程序，而是运行由 Go 程序推导出的 ELF 文件所代表的 MIPS 程序，在链上也实现了 MIPS 指令集，二者可以保持等效。

MIPS 是一种简单指令集操作系统，使得在 Solidity 中的实现成为可能。

MIPS 主要包含两种类型的指令：
1. 常规指令，用于执行常规的程序计算和控制，如算术运算、数据加载、条件分支等。
2. 系统指令，syscall 用于执行系统调用，即请求操作系统提供的服务，如读取文件、创建进程等。特别地，读取操作需要 Pre-image 合约的配合，如读取 block headers、MPT nodes、receipts、transactions、blobs 等数据。

两种关键类型的数据结构包括：
1. State（状态）

```
    struct State {
        bytes32 memRoot;
        bytes32 preimageKey;
        uint32 preimageOffset;
        uint32 pc;
        uint32 nextPC;
        uint32 lo;
        uint32 hi;
        uint32 heap;
        uint8 exitCode;
        bool exited;
        uint64 step;
        uint32[32] registers;
    }
```

链上 MIPS 只是一个纯逻辑函数，不包含任何状态，所有状态都是通过调用时传入的。State 类型主要包含内存信息、指令信息、寄存器信息等。

2. ProofData

ProofData 是[默克尔树证明数据](https://medium.com/crypto-0-nite/merkle-proofs-explained-6dd429623dc5)的紧密排列，用于证明 state 数据的有效性。

如果你对 MIPS 的完整细节感兴趣，可以参考[这里](https://docs.optimism.io/stack/protocol/fault-proofs/mips#further-reading)。

## pre-image-oracle

如果将一个 transaction 的执行拆解为一系列指令，其中大部分是基础指令，少部分是系统指令，而在系统指令中，只有大约 0.1% 是需要进行读取操作的系统指令。因此，在设计 STEP() 函数时，需要将这两种情况分开处理，基础指令所需的上下文直接通过调用时的入参传入。而需要读取特殊信息的系统指令，考虑到其出现的概率很低，出于设计考虑，不占用入参，而是为其设计了一个单独的 Pre-image-oracle 组件来传递相应的数据，这就是 pre-image 在 fault-proof 中的作用，起到一个通讯中介的角色。

## CANNON 在链上的体现

CANNON 在链上的实体为 MIPS.sol 文件，其中最主要的部分为 [step() 函数](https://github.com/ethereum-optimism/optimism/blob/develop/packages/contracts-bedrock/src/cannon/MIPS.sol#L204)，此函数由我们之前章节的 FDG 中的 step() 函数直接调用。

以下是 `step()` 函数的完整代码解析，主要分为以下几部分逻辑：

1. 数据校验：在解包数据前进行内存偏移和结构完整性的检查。
2. 定义辅助函数：用于将 state 数据紧密存储在内存中，便于后续使用。
3. 获取指令内容：通过 state 中的 pc 来获取具体的 instruction 内容，并在此过程中使用默克尔树来验证 state 数据间的关系，确保数据的正确性。
4. 系统指令调用：如果是 syscall（系统指令调用），则跳到相应的处理逻辑中执行并返回结果。如果是读取操作，需要额外与 Pre-image-oracle 交互。
5. 基础指令执行：执行基础指令，并更新相关变量后返回结果。

对于上述第 3、4 和 5 步骤，此处不再进行额外讲解，但如果您对这些部分感兴趣，可以通过以下链接直接访问相关代码进行深入阅读：
- 第 3 步：[详细代码](https://github.com/ethereum-optimism/optimism/blob/5e317379fae65b76f5a6ee27581f0e62d2fe017a/packages/contracts-bedrock/src/cannon/libraries/MIPSInstructions.sol#L14)
- 第 4 步：[详细代码](https://github.com/ethereum-optimism/optimism/blob/5e317379fae65b76f5a6ee27581f0e62d2fe017a/packages/contracts-bedrock/src/cannon/MIPS.sol#L134)
- 第 5 步：[详细代码](https://github.com/ethereum-optimism/optimism/blob/5e317379fae65b76f5a6ee27581f0e62d2fe017a/packages/contracts-bedrock/src/cannon/libraries/MIPSInstructions.sol#L41)

```
   function step(bytes calldata _stateData, bytes calldata _proof, bytes32 _localContext) public returns (bytes32) {
        unchecked {
            State memory state;
            //-------------------- Part 1 start --------------------
            // Packed calldata is ~6 times smaller than state size
            assembly {
                if iszero(eq(state, 0x80)) {
                    // expected state mem offset check
                    revert(0, 0)
                }
                if iszero(eq(mload(0x40), shl(5, 48))) {
                    // expected memory check
                    revert(0, 0)
                }
                if iszero(eq(_stateData.offset, 132)) {
                    // 32*4+4=132 expected state data offset
                    revert(0, 0)
                }
                if iszero(eq(_proof.offset, STEP_PROOF_OFFSET)) {
                    // 132+32+256=420 expected proof offset
                    revert(0, 0)
                }
            //-------------------- Part 1 end ----------------------
            
            //-------------------- Part 2 start --------------------
                function putField(callOffset, memOffset, size) -> callOffsetOut, memOffsetOut {
                    // calldata is packed, thus starting left-aligned, shift-right to pad and right-align
                    let w := shr(shl(3, sub(32, size)), calldataload(callOffset))
                    mstore(memOffset, w)
                    callOffsetOut := add(callOffset, size)
                    memOffsetOut := add(memOffset, 32)
                }

                // Unpack state from calldata into memory
                let c := _stateData.offset // calldata offset
                let m := 0x80 // mem offset
                c, m := putField(c, m, 32) // memRoot
                c, m := putField(c, m, 32) // preimageKey
                c, m := putField(c, m, 4) // preimageOffset
                c, m := putField(c, m, 4) // pc
                c, m := putField(c, m, 4) // nextPC
                c, m := putField(c, m, 4) // lo
                c, m := putField(c, m, 4) // hi
                c, m := putField(c, m, 4) // heap
                c, m := putField(c, m, 1) // exitCode
                c, m := putField(c, m, 1) // exited
                c, m := putField(c, m, 8) // step

                // Unpack register calldata into memory
                mstore(m, add(m, 32)) // offset to registers
                m := add(m, 32)
                for { let i := 0 } lt(i, 32) { i := add(i, 1) } { c, m := putField(c, m, 4) }
            }
            //-------------------- Part 2 end ----------------------
            
            // Don't change state once exited
            if (state.exited) {
                return outputState();
            }

            state.step += 1;
            
            //-------------------- Part 3 start --------------------
            // instruction fetch
            uint256 insnProofOffset = MIPSMemory.memoryProofOffset(STEP_PROOF_OFFSET, 0);
            (uint32 insn, uint32 opcode, uint32 fun) =
                ins.getInstructionDetails(state.pc, state.memRoot, insnProofOffset);
            //-------------------- Part 3 end ----------------------

            //-------------------- Part 4 start --------------------
            // Handle syscall separately
            // syscall (can read and write)
            if (opcode == 0 && fun == 0xC) {
                return handleSyscall(_localContext);
            }
            //-------------------- Part 4 end ----------------------
            
            //-------------------- Part 5 start --------------------

            // Exec the rest of the step logic
            st.CpuScalars memory cpu = getCpuScalars(state);
            (state.memRoot) = ins.execMipsCoreStepLogic({
                _cpu: cpu,
                _registers: state.registers,
                _memRoot: state.memRoot,
                _memProofOffset: MIPSMemory.memoryProofOffset(STEP_PROOF_OFFSET, 1),
                _insn: insn,
                _opcode: opcode,
                _fun: fun
            });
            setStateCpuScalars(state, cpu);

            return outputState();
            //-------------------- Part 5 end ----------------------
        }
    }
```

## CANNON 在链下的体现

CANNON 在链下主要体现在 [Cannon](https://github.com/ethereum-optimism/optimism/tree/develop/cannon) 组件中，该组件可用于生成指令的单独执行流程，或持续执行并在执行过程中产生输出。

### 主要内容

执行并提供游戏中 move/step 的输入参数内容。

#### 执行
当我们执行 `cannon run -h` 时，可以看到运行时需要传入的 flag。通过这些 flag，我们可以深入理解 Cannon 链下执行的机制。

```
./bin/cannon run -h
NAME:
   cannon run - Run VM step(s) and generate proof data to replicate onchain.

USAGE:
   cannon run [command options] [arguments...]

DESCRIPTION:
   Run VM step(s) and generate proof data to replicate onchain. See flags to match when to output a proof, a snapshot, or to stop early.

OPTIONS:
   --type value                          VM type to run. Options are 'cannon' (default) (default: "cannon")
   --input value                         path of input JSON state. Stdin if left empty. (default: "state.json")
   --output value                        path of output JSON state. Not written if empty, use - to write to Stdout. (default: "out.json")
   --proof-at value                      step pattern to output proof at: 'never' (default), 'always', '=123' at exactly step 123, '%123' for every 123 steps
   --proof-fmt value                     format for proof data output file names. Proof data is written to stdout if -. (default: "proof-%d.json")
   --snapshot-at value                   step pattern to output snapshots at: 'never' (default), 'always', '=123' at exactly step 123, '%123' for every 123 steps
   --snapshot-fmt value                  format for snapshot output file names. (default: "state-%d.json")
   --stop-at value                       step pattern to stop at: 'never' (default), 'always', '=123' at exactly step 123, '%123' for every 123 steps
   --stop-at-preimage value              stop at the first preimage request matching this key
   --stop-at-preimage-type value         stop at the first preimage request matching this type
   --stop-at-preimage-larger-than value  stop at the first step that requests a preimage larger than the specified size (in bytes)
   --meta value                          path to metadata file for symbol lookup for enhanced debugging info during execution. (default: "meta.json")
   --info-at value                       step pattern to print info at: 'never' (default), 'always', '=123' at exactly step 123, '%123' for every 123 steps (default: %100000)
   --pprof.cpu                           enable pprof cpu profiling (default: false)
   --debug                               enable debug mode, which includes stack traces and other debug info in the output. Requires --meta. (default: false)
   --debug-info value                    path to write debug info to
   --help, -h                            show help
```

以下是几个核心的 flag：
- `type`: 指的是 VM 的类型，目前仅支持 cannon，未来将支持更多类型的虚拟机。
- `input`: 指的是 State 类型的 JSON 形式文件的路径，由 ELF 文件加载而来，可以理解为 client 代码在运行时的环境，如内存分布、寄存器等的状态。
- `output`: 根据 input 执行后的最新状态的输出路径。
- `proof-at`: 运行到 x 位置后输出 proof。
- `snapshot-at`: 运行到 x 位置后输出 state。
- `stop-at`: 在第 x 次 step 后终止。

Cannon 的命令实际上并不是为手动单次执行设计的，而是为了 op-challenger 而设计。让我们看一下 op-challenger 中是如何使用的。

```
func (e *Executor) GenerateProof(ctx context.Context, dir string, i uint64) error {
	snapshotDir := filepath.Join(dir, snapsDir)
	start, err := e.selectSnapshot(e.logger, snapshotDir, e.absolutePreState, i)
	if err != nil {
		return fmt.Errorf("find starting snapshot: %w", err)
	}
	proofDir := filepath.Join(dir, proofsDir)
	dataDir := filepath.Join(dir, preimagesDir)
	lastGeneratedState := filepath.Join(dir, finalState)
	args := []string{
		"run",
		"--input", start,
		"--output", lastGeneratedState,
		"--meta", "",
		"--info-at", "%" + strconv.FormatUint(uint64(e.infoFreq), 10),
		"--proof-at", "=" + strconv.FormatUint(i, 10),
		"--proof-fmt", filepath.Join(proofDir, "%d.json.gz"),
		"--snapshot-at", "%" + strconv.FormatUint(uint64(e.snapshotFreq), 10),
		"--snapshot-fmt", filepath.Join(snapshotDir, "%d.json.gz"),
	}
	if i < math.MaxUint64 {
		args = append(args, "--stop-at", "="+strconv.FormatUint(i+1, 10))
	}
```

可以看到，它是在 GenerateProof 函数下使用的，而这个函数主要由 move（attack & defend）和 step 使用，即每次 move 和 step 前都会调用 Cannon。我们继续看一下它如何向 flag 中传值，可以看到 start 来自第 i 步操作，并且 cannon 执行到 i+1 处停止，因此整个 cannon 的执行只进行了一步。

因此，我们需要纠正一个常见误区：cannon 在链下的虚拟机并不是持续运行的，而是按上述方式单次运行以获取特定位置的数据。

#### 具体实现
链下 cannon 执行时的核心组件在于 cannon 中的 [Step()](https://github.com/ethereum-optimism/optimism/blob/develop/cannon/mipsevm/singlethreaded/instrumented.go#L66) 函数和其中的 [mipsStep()](https://github.com/ethereum-optimism/optimism/blob/develop/cannon/mipsevm/singlethreaded/mips.go#L50) 函数。

可以看到，链下的 Go 代码中的 step 逻辑与链上 Solidity 的 step 逻辑高度一致，首先将内存等状态加载到固定位置，然后在 mipsStep() 中进一步处理，例如判定是否为 syscall 等操作。


```
func (m *InstrumentedState) Step(proof bool) (wit *mipsevm.StepWitness, err error) {
	m.preimageOracle.Reset()
	m.memoryTracker.Reset(proof)

	if proof {
		insnProof := m.state.Memory.MerkleProof(m.state.Cpu.PC)
		encodedWitness, stateHash := m.state.EncodeWitness()
		wit = &mipsevm.StepWitness{
			State:     encodedWitness,
			StateHash: stateHash,
			ProofData: insnProof[:],
		}
	}
	err = m.mipsStep()
	if err != nil {
		return nil, err
	}

	if proof {
		memProof := m.memoryTracker.MemProof()
		wit.ProofData = append(wit.ProofData, memProof[:]...)
		lastPreimageKey, lastPreimage, lastPreimageOffset := m.preimageOracle.LastPreimage()
		if lastPreimageOffset != ^uint32(0) {
			wit.PreimageOffset = lastPreimageOffset
			wit.PreimageKey = lastPreimageKey
			wit.PreimageValue = lastPreimage
		}
	}
	return
}
```
```
func (m *InstrumentedState) mipsStep() error {
	if m.state.Exited {
		return nil
	}
	m.state.Step += 1
	// instruction fetch
	insn, opcode, fun := exec.GetInstructionDetails(m.state.Cpu.PC, m.state.Memory)

	// Handle syscall separately
	// syscall (can read and write)
	if opcode == 0 && fun == 0xC {
		return m.handleSyscall()
	}

	// Exec the rest of the step logic
	return exec.ExecMipsCoreStepLogic(&m.state.Cpu, &m.state.Registers, m.state.Memory, insn, opcode, fun, m.memoryTracker, m.stackTracker)
}
```

## 总结

经过上述详细讲解，我们了解到链上部分主要用于 Fault Proof Game 的最细粒度验证，而链下部分则旨在逐步推导出相应的最细粒度位置的数据，供链上执行使用。如果细致观察，可以发现二者的架构模式基本一致，但仍存在细微差别。链上的 MIPS 系统主要用于验证，因此只需关注结果；而链下的 MIPS 系统则更注重于产生可用的数据，其在数据输出和存储方面更加友好和高效。

通过这种设计，CANNON 在确保链上与链下数据一致性的同时，也优化了数据处理和验证过程，使整个系统的运行更加高效和可靠。这种双层验证机制为 Fault Proof Game 提供了坚实的技术支持，确保了游戏的公平性和透明性。
